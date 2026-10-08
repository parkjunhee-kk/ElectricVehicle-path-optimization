# EV Route Optimization with Charging Decisions

전기차가 출발지에서 도착지까지 이동할 때 **어느 충전소를 경유하고 얼마나 충전할지**를 함께 결정해, 주행 시간·충전 시간·충전 비용의 가중합을 최소화하는 경로 최적화 프로젝트입니다. Google OR-Tools의 Min-Cost Flow와 MILP(CBC)를 사용했습니다.

> 최적화 수업 팀 프로젝트

<p align="center"><img src="assets/mcf_route.png" width="640"></p>

## Problem Setting

| 파라미터 | 값 |
|---|---|
| 배터리 용량 | 70 kWh (출발 시 완충) |
| 주행 에너지 소모 | 0.15 kWh/km |
| 최소 잔량 | 5.6 kWh (8%) |
| 주행 속력 | 90 km/h |
| 목적함수 가중치 | 주행시간 1.0 · 충전시간 1.0 · 충전비용 0.1 |

**데이터**: 전 세계 5,000개 충전소 데이터(`data/detailed_ev_charging_stations.csv`)에서 위도, 경도, 충전 속도(kW), 충전 단가(USD/kWh)를 사용합니다. 충전소 간 거리는 Haversine 공식으로 계산합니다. 충전소가 대륙 단위로 흩어져 있어 원거리 간선 때문에 그래프가 끊기는 문제가 생기므로, 거리를 $55\cdot\log(1+d)$로 스케일링해 사용했습니다.

## Methods

### 1. Min-Cost Flow (전체 5,000개 충전소)

간선 $(i,j)$의 비용을 다음과 같이 두고, 출발지에서 1단위 유량을 보내는 최소비용 유량 문제로 최단 경로를 구합니다.

$$
c_{ij} = w_d\, t_{ij} + w_{ct}\,\frac{e_{ij}}{P_i} + w_{cc}\, p_i\, e_{ij}
$$

여기서 $t_{ij}$는 주행 시간, $e_{ij}$는 구간 소모 에너지, $P_i$는 충전 속도, $p_i$는 충전 단가입니다. 사용 가능 에너지(70 − 5.6 kWh)를 넘는 간선은 제거하고, 각 충전소에서는 다음 구간에 필요한 만큼 충전한다고 가정합니다.

### 2. Baseline: Greedy

- **재방문 허용**: 매 스텝 비용이 가장 작은 간선 선택 → 같은 노드 사이를 순환하며 도착 실패
- **재방문 금지**: 도착은 하지만 2,821개 구간을 거치는 매우 비효율적인 경로

### 3. MILP with Battery Dynamics (15개 노드)

배터리 잔량과 충전량을 연속 변수로 두는 MILP입니다. 5,000개 노드 전체에 대해서는 계산이 불가능해 15개 충전소 부분집합(`data/stations_subset_15.xlsx`)에서 풉니다.

$$
\min \sum_{i\ne j} x_{ij}\, w_d\, t_{ij} + \sum_i Q_i\left(\frac{w_{ct}}{P_i} + w_{cc}\, p_i\right)
$$

$$
\begin{aligned}
&\textstyle\sum_j x_{ij} - \sum_j x_{ji} = \mathbb{1}[i=s] - \mathbb{1}[i=t] && \text{(flow conservation)}\\
&E_s = C,\quad E_{\min} \le E_i \le C,\quad E_i + Q_i \le C && \text{(battery limits)}\\
&|E_j - (E_i + Q_i - e_{ij})| \le M(1 - x_{ij}) && \text{(energy dynamics, Big-M)}\\
&x_{ij} \in \{0,1\},\quad Q_i \ge 0
\end{aligned}
$$

$E_i$는 도착 시 잔량, $Q_i$는 충전량, $C$는 배터리 용량입니다.

<p align="center"><img src="assets/milp_route_15nodes.png" width="560"></p>

### 4. Discrete SOC State Graph (전체 5,000개 충전소)

배터리 잔량을 {0, 30, 50, 70, 100}%로 이산화하고 (충전소, SOC 단계)를 상태 노드로 하는 확장 그래프를 만듭니다. 주행 간선과 충전 간선을 함께 넣어 Min-Cost Flow로 풀어, 충전량 결정을 전체 네트워크로 확장합니다.

<p align="center"><img src="assets/discrete_soc_route.png" width="560"></p>

## Results

| Method | 노드 수 | 결과 | 목적함수 |
|---|---:|---|---:|
| Min-Cost Flow | 5,000 | 9개 충전소 경유 (10구간) | **64.78** |
| Greedy (재방문 허용) | 5,000 | 10,000 스텝 내 도착 실패 | – |
| Greedy (재방문 금지) | 5,000 | 2,821구간 | 3,323.86 |
| Discrete SOC | 5,000 | 8개 충전소 경유, 충전 9.36h · $138.95 | 63.58* |
| MILP + 배터리 동역학 | 15 | 0→12→13→1→10, 주행 17.60h · 충전 0.91h · $42.56 | 22.76 |

\* Discrete SOC는 충전 시간을 로그 스케일로 반영하고 충전량이 이산적이라 다른 방법과 목적함수 값을 직접 비교할 수 없습니다.

## Repository Structure

```
├── EV_route_optimization.ipynb
├── data/
│   ├── detailed_ev_charging_stations.csv   # 전체 충전소 5,000개
│   └── stations_subset_15.xlsx             # MILP용 15개 충전소 (ID 9~23)
├── assets/                                 # README 그림
└── README.md
```

## How to Run

Google Colab에서 작성되었습니다. `data/` 폴더의 파일을 노트북과 같은 위치(또는 Colab `/content/`)에 두고 실행하면 됩니다.

```bash
pip install ortools haversine numpy pandas matplotlib seaborn openpyxl
```

전체 5,000개 노드의 거리 행렬과 확장 그래프를 메모리에 올리므로 RAM 12GB 이상(Colab 기본 런타임)을 권장합니다.

## Limitations

- 거리를 로그 스케일로 변환했기 때문에 주행 시간과 에너지는 실제 물리량이 아닌 상대적인 값입니다.
- 대기 시간, 충전소 혼잡도, 운영 시간(Availability)은 반영하지 않았습니다.
- MILP는 노드 수가 늘면 계산량이 급격히 증가해 15개 노드로 제한했습니다.
