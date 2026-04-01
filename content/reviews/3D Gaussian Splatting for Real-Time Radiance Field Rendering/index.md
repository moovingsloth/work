---
title: "3D Gaussian Splatting for Real-Time Radiance Field Rendering"
date: 2026-04-01
tags:
  - 3DGS
featured: true
weight: 50
insight: "Neural radiance field의 implicit 표현을 explicit 3D Gaussian primitives로 대체하고, 미분 가능한 타일 기반 래스터라이저를 통해 실시간 고품질 novel view synthesis를 달성한 논문."
summary: "SfM 초기화부터 공분산 행렬 파라미터화, 적응적 밀도 제어, 타일 기반 스플래팅까지 — 3D Gaussian Splatting의 전체 파이프라인을 수식과 수도코드 수준에서 분석한다."
---

## 논문의 위치: NeRF 패러다임의 전환점

NeRF(2020)는 연속적인 5D 함수 $F_\Theta : (\mathbf{x}, \mathbf{d}) \to (\mathbf{c}, \sigma)$를 MLP로 학습하여 novel view synthesis의 품질을 극적으로 끌어올렸다. 그러나 단일 픽셀을 렌더링하기 위해 ray 위의 수십~수백 개 샘플 포인트에서 MLP를 평가해야 하므로, 실시간 렌더링은 원천적으로 불가능했다.

이후 Instant-NGP, Plenoxels, TensoRF 등이 MLP를 voxel grid나 hash table로 대체하여 학습 속도를 개선했지만, 렌더링은 여전히 volumetric ray marching에 의존했다. **3D Gaussian Splatting(3DGS)**은 이 구조 자체를 버린다. 장면을 수백만 개의 3D 가우시안으로 표현하고, ray marching 대신 **래스터화(rasterization)** 기반으로 렌더링함으로써, 학습 품질과 실시간 렌더링을 동시에 달성한 최초의 방법이다.

---

## 1. Structure from Motion (SfM) 초기화

### SfM이 왜 필요한가

3DGS는 장면을 명시적 포인트 집합으로 표현하므로, 학습 시작 시 가우시안의 초기 위치가 필요하다. 이를 위해 COLMAP 기반의 SfM 파이프라인을 사용한다.

**SfM 파이프라인 요약:**
1. 입력 이미지들에서 SIFT 등의 특징점을 추출한다.
2. 이미지 쌍 간 특징점 매칭 후, fundamental matrix를 추정하여 기하학적으로 일관된 매칭만 남긴다.
3. Incremental SfM: 초기 이미지 쌍에서 삼각측량으로 3D 포인트를 복원하고, 새로운 이미지를 PnP로 등록하면서 bundle adjustment를 반복한다.
4. 최종 출력: **sparse point cloud** $\mathcal{P} = \{(\mathbf{p}_i, \mathbf{c}_i)\}$ 와 각 이미지의 카메라 파라미터 $\{(K_j, [R_j | \mathbf{t}_j])\}$.

### 초기화 과정

SfM에서 얻은 각 3D 포인트 $\mathbf{p}_i$에 대해 하나의 가우시안을 생성한다:

- **위치** $\boldsymbol{\mu}_i = \mathbf{p}_i$
- **공분산**: 가장 가까운 이웃 3개 포인트까지의 평균 거리를 반지름으로 하는 등방성(isotropic) 가우시안으로 초기화
- **색상**: SfM에서 얻은 RGB 값으로 SH 계수의 0차 항(DC term)을 초기화하고, 고차 항은 0으로 설정
- **불투명도** $\alpha_i$: 작은 양수(예: 0.1)로 초기화

> **핵심 포인트:** SfM point cloud는 수천~수만 개로 sparse하지만, 이후 Adaptive Density Control에 의해 수백만 개로 증가한다. 초기화의 역할은 최적화의 출발점을 기하학적으로 합리적인 위치에 놓는 것이다.

---

## 2. 가우시안 표현과 공분산 행렬

### 3D 가우시안의 정의

각 가우시안 $G_i$는 다음 파라미터로 정의된다:

$$G_i(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu}_i)^T \boldsymbol{\Sigma}_i^{-1} (\mathbf{x} - \boldsymbol{\mu}_i)}$$

여기서:
- $\boldsymbol{\mu}_i \in \mathbb{R}^3$ : 가우시안의 중심 위치
- $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ : 공분산 행렬 (양의 반정치 행렬)
- $\alpha_i \in [0, 1]$ : 불투명도 (sigmoid 활성화로 제한)
- $\mathbf{sh}_i$ : 구면 조화 계수 (뷰 의존적 색상)

### 공분산 행렬의 파라미터화

공분산 행렬은 반드시 **양의 반정치(positive semi-definite)** 여야 한다. 임의의 $3 \times 3$ 행렬을 직접 최적화하면 이 조건이 깨질 수 있으므로, 논문은 다음과 같은 분해를 사용한다:

$$\boldsymbol{\Sigma} = R S S^T R^T$$

여기서:
- $R \in SO(3)$: 회전 행렬, **단위 쿼터니언** $\mathbf{q} = (q_w, q_x, q_y, q_z)$로 파라미터화
- $S = \text{diag}(s_x, s_y, s_z)$: 스케일 행렬, 각 축 방향의 표준편차

이 분해의 핵심적 이점:
1. **양의 반정치 보장**: $S S^T$는 항상 양의 반정치이고, 회전은 이 성질을 보존한다.
2. **직관적 해석**: 각 가우시안은 3D 공간에서 특정 방향($R$)으로 특정 크기($S$)만큼 늘어난 타원체이다.
3. **기울기 전파**: $\mathbf{q}$와 $(s_x, s_y, s_z)$에 대한 기울기가 $\boldsymbol{\Sigma}$를 거쳐 역전파된다.

쿼터니언에서 회전 행렬로의 변환:

$$R = \begin{pmatrix} 1-2(q_y^2+q_z^2) & 2(q_xq_y-q_wq_z) & 2(q_xq_z+q_wq_y) \\ 2(q_xq_y+q_wq_z) & 1-2(q_x^2+q_z^2) & 2(q_yq_z-q_wq_x) \\ 2(q_xq_z-q_wq_y) & 2(q_yq_z+q_wq_x) & 1-2(q_x^2+q_y^2) \end{pmatrix}$$

---

## 3. Radiance Field로서의 가우시안

### NeRF의 Radiance Field와의 비교

NeRF는 **연속적 volumetric radiance field**를 정의한다:

$$C(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \, \sigma(\mathbf{r}(t)) \, \mathbf{c}(\mathbf{r}(t), \mathbf{d}) \, dt, \quad T(t) = \exp\left(-\int_{t_n}^{t} \sigma(\mathbf{r}(s)) \, ds\right)$$

3DGS는 이를 **이산적 가우시안 혼합(Gaussian mixture)**으로 대체한다. 각 가우시안이 공간의 특정 영역에서 밀도와 색상을 동시에 기술하므로, 가우시안 집합 전체가 하나의 radiance field를 구성한다.

### 구면 조화 함수 (Spherical Harmonics)

뷰 의존적 색상을 표현하기 위해, 각 가우시안은 구면 조화(SH) 계수를 저장한다. 차수 $l$까지의 SH를 사용하면 $(l+1)^2$개의 계수가 필요하다.

- **$l = 0$**: 1개 계수 (뷰 독립적, Lambertian)
- **$l = 1$**: 4개 계수 (기본적인 방향 의존성)
- **$l = 2$**: 9개 계수 (거울 반사 등 표현 가능)
- **$l = 3$**: 16개 계수 (논문의 기본 설정)

방향 $\mathbf{d} = (\theta, \phi)$에서의 색상:

$$c(\mathbf{d}) = \sum_{l=0}^{l_{\max}} \sum_{m=-l}^{l} k_l^m \, Y_l^m(\mathbf{d})$$

여기서 $k_l^m$이 학습되는 SH 계수이고, $Y_l^m$은 SH 기저 함수이다. RGB 각 채널에 대해 독립적인 SH 계수를 갖는다.

---

## 4. 2D로의 투영과 스플래팅 (Splatting)

### 3D에서 2D로의 가우시안 투영

3D 가우시안을 이미지 평면에 렌더링하려면, 카메라 좌표계로 변환 후 2D로 투영해야 한다. Zwicker et al.(2001)의 EWA(Elliptical Weighted Average) splatting 이론에 따라:

**1단계 — 월드에서 카메라 좌표계로:**

뷰 변환 행렬 $W = [R_{\text{cam}} | \mathbf{t}_{\text{cam}}]$를 적용하면, 카메라 좌표계에서의 가우시안 중심과 공분산은:

$$\boldsymbol{\mu}' = W \boldsymbol{\mu}, \quad \boldsymbol{\Sigma}' = W \boldsymbol{\Sigma} W^T$$

**2단계 — 카메라에서 이미지 평면으로 (투영):**

원근 투영은 비선형이므로, 가우시안 중심 근방에서 **야코비안(Jacobian)** $J$를 이용한 1차 근사를 사용한다:

$$J = \begin{pmatrix} \frac{f_x}{z} & 0 & -\frac{f_x \cdot x}{z^2} \\ 0 & \frac{f_y}{z} & -\frac{f_y \cdot y}{z^2} \end{pmatrix}$$

결과적으로, 이미지 평면에서의 **2D 공분산 행렬**은:

$$\boldsymbol{\Sigma}^{2D} = J W \boldsymbol{\Sigma} W^T J^T$$

이 $2 \times 2$ 행렬이 화면상의 타원형 가우시안 풋프린트를 정의한다.

### 알파 합성 (Alpha Compositing)

픽셀 $\mathbf{p}$의 최종 색상은, 해당 픽셀에 겹치는 가우시안들을 깊이 순으로 정렬한 뒤 front-to-back 알파 블렌딩으로 계산한다:

$$C(\mathbf{p}) = \sum_{i \in \mathcal{N}} \mathbf{c}_i \, \alpha'_i \prod_{j=1}^{i-1}(1 - \alpha'_j)$$

여기서 $\alpha'_i$는 가우시안 $i$의 불투명도 $\alpha_i$에 해당 픽셀 위치에서의 2D 가우시안 값을 곱한 것이다:

$$\alpha'_i = \alpha_i \cdot \exp\left( -\frac{1}{2} (\mathbf{p} - \boldsymbol{\mu}_i^{2D})^T (\boldsymbol{\Sigma}_i^{2D})^{-1} (\mathbf{p} - \boldsymbol{\mu}_i^{2D}) \right)$$

---

## 5. 타일 기반 래스터라이저

3DGS의 실시간 성능의 핵심은 **타일 기반(tile-based) 래스터화** 파이프라인이다.

### 파이프라인 구조

```
입력 가우시안 집합
    │
    ▼
[1] Frustum Culling ─── 뷰 절두체 밖의 가우시안 제거
    │
    ▼
[2] 2D 투영 ─── 3D → 2D 공분산 변환, 화면 좌표 계산
    │
    ▼
[3] 타일 할당 ─── 각 가우시안이 겹치는 16×16 타일 결정
    │                 하나의 가우시안이 여러 타일에 할당될 수 있음
    ▼
[4] 키 생성 ─── (tile_id, depth) 조합의 64비트 키 생성
    │
    ▼
[5] GPU Radix Sort ─── 키 기준 정렬 → 타일별로 깊이 순서 보장
    │
    ▼
[6] 타일별 렌더링 ─── 각 타일을 하나의 CUDA 블록이 처리
                        블록 내 스레드가 각 픽셀 담당
                        공유 메모리에 가우시안 배치 로드
                        front-to-back 알파 블렌딩
                        누적 투명도 < ε 이면 조기 종료
```

### 왜 타일 기반인가

- **정렬 효율**: 가우시안을 전역적으로 한 번만 정렬하면, 모든 타일에서 깊이 순서가 보장된다.
- **병렬성**: 각 타일이 독립적으로 처리되므로 GPU 활용도가 극대화된다.
- **메모리 지역성**: 16×16 타일 단위로 처리하면 공유 메모리를 효과적으로 활용할 수 있다.
- **역전파 호환**: 정렬 순서가 결정적(deterministic)이므로, forward와 동일한 순서로 backward 패스를 실행할 수 있다.

---

## 6. 적응적 밀도 제어 (Adaptive Density Control)

SfM 초기화만으로는 장면의 세밀한 디테일을 표현하기에 가우시안 수가 부족하다. ADC는 학습 중에 가우시안의 분포를 동적으로 조정한다.

### 세 가지 연산

**Clone (복제):** 위치 기울기의 크기가 임계값 $\tau_{\text{pos}}$를 초과하고, 가우시안의 스케일이 작은 경우 — under-reconstruction 영역. 동일한 가우시안을 복사하여 기울기 방향으로 이동시킨다.

**Split (분할):** 위치 기울기의 크기가 임계값 $\tau_{\text{pos}}$를 초과하고, 가우시안의 스케일이 큰 경우 — over-reconstruction 영역. 가우시안을 두 개의 작은 가우시안으로 분할한다. 새 가우시안은 원래 가우시안의 PDF를 샘플링하여 위치를 결정하고, 스케일을 $1/\phi$ ($\phi = 1.6$)로 축소한다.

**Prune (제거):** 불투명도 $\alpha_i$가 임계값 $\epsilon_\alpha$ 이하인 가우시안을 제거한다. 또한 주기적으로 모든 가우시안의 불투명도를 낮은 값으로 리셋하여, 불필요한 가우시안이 자연스럽게 사라지도록 한다.

### ADC 수도코드

```python
def adaptive_density_control(gaussians, grad_accum, iteration):
    """
    gaussians: 현재 가우시안 집합
        - μ: 위치 [N, 3]
        - Σ (q, s): 공분산 파라미터
        - α: 불투명도 [N]
        - sh: SH 계수
    grad_accum: 위치에 대한 기울기 누적값 [N]
    """
    # 매 100 iteration마다 ADC 실행
    if iteration % 100 != 0:
        return gaussians
    
    avg_grad = grad_accum / count  # 누적 기울기의 평균
    
    # --- Prune: 투명한 가우시안 제거 ---
    mask_prune = gaussians.α < ε_α  # ε_α ≈ 0.005
    gaussians.remove(mask_prune)
    
    # --- 위치 기울기가 큰 가우시안 식별 ---
    mask_large_grad = avg_grad > τ_pos  # τ_pos = 0.0002
    
    # --- Clone: 작은 가우시안 복제 ---
    mask_clone = mask_large_grad & (gaussians.scale < τ_scale)
    new_gaussians = gaussians[mask_clone].copy()
    new_gaussians.μ += grad_direction  # 기울기 방향으로 이동
    gaussians.add(new_gaussians)
    
    # --- Split: 큰 가우시안 분할 ---
    mask_split = mask_large_grad & (gaussians.scale >= τ_scale)
    for g in gaussians[mask_split]:
        g1, g2 = split_gaussian(g, scale_factor=1/1.6)
        # g1, g2의 위치는 원래 가우시안의 PDF에서 샘플링
        gaussians.replace(g, [g1, g2])
    
    # --- 주기적 불투명도 리셋 (매 3000 iteration) ---
    if iteration % 3000 == 0:
        gaussians.α = sigmoid_inv(0.01)  # 낮은 값으로 리셋
    
    return gaussians
```

---

## 7. 전체 최적화 파이프라인

### 손실 함수

$$\mathcal{L} = (1 - \lambda) \, \mathcal{L}_1 + \lambda \, \mathcal{L}_{\text{D-SSIM}}$$

- $\mathcal{L}_1$: 렌더링 이미지와 GT 이미지 간의 L1 손실
- $\mathcal{L}_{\text{D-SSIM}}$: 구조적 유사도 손실 (1 - SSIM)
- $\lambda = 0.2$ (논문 기본값)

### 최적화 수도코드

```python
def train_3dgs(images, cameras, sfm_points, num_iterations=30000):
    """
    전체 3D Gaussian Splatting 학습 파이프라인
    """
    # ===== 초기화 =====
    gaussians = initialize_from_sfm(sfm_points)
    # 각 포인트 → (μ, q, s, α, sh) 파라미터 생성
    # q: 단위 쿼터니언 [1,0,0,0]
    # s: KNN 기반 초기 스케일
    # α: sigmoid_inv(0.1)
    # sh: DC항만 SfM 색상으로, 나머지 0
    
    optimizer = Adam([
        {'params': gaussians.μ,  'lr': 1.6e-4 → 1.6e-6},  # 위치 (지수 감쇠)
        {'params': gaussians.q,  'lr': 1e-3},               # 회전
        {'params': gaussians.s,  'lr': 5e-3},               # 스케일
        {'params': gaussians.α,  'lr': 5e-2},               # 불투명도
        {'params': gaussians.sh, 'lr': 2.5e-3 / 12.5e-4},  # SH (DC / 고차)
    ])
    
    grad_accum = zeros(len(gaussians))  # 위치 기울기 누적
    
    for iter in range(num_iterations):
        # ===== 1. 랜덤 뷰 선택 =====
        cam = random_camera(cameras)
        gt_image = images[cam.id]
        
        # ===== 2. 렌더링 (미분 가능한 래스터라이저) =====
        # 2a. 공분산 구성: Σ = R(q) · S · S^T · R(q)^T
        # 2b. 2D 투영: Σ_2d = J · W · Σ · W^T · J^T
        # 2c. 타일 기반 정렬 + 알파 블렌딩
        rendered = rasterize(gaussians, cam)
        
        # ===== 3. 손실 계산 =====
        loss = (1 - λ) * L1(rendered, gt_image) + λ * D_SSIM(rendered, gt_image)
        
        # ===== 4. 역전파 =====
        loss.backward()  # ∂L/∂μ, ∂L/∂q, ∂L/∂s, ∂L/∂α, ∂L/∂sh 계산
        
        # ===== 5. 위치 기울기 누적 (ADC용) =====
        grad_accum += ||∂L/∂μ||  # 스크린 스페이스 위치 기울기의 norm
        
        # ===== 6. 파라미터 업데이트 =====
        optimizer.step()
        optimizer.zero_grad()
        
        # ===== 7. 적응적 밀도 제어 =====
        if iter >= 500 and iter <= 15000:
            adaptive_density_control(gaussians, grad_accum, iter)
            grad_accum.reset()  # ADC 실행 후 기울기 누적 초기화
    
    return gaussians
```

### 학습률 스케줄 상세

위치 $\boldsymbol{\mu}$에는 **지수 감쇠(exponential decay)** 스케줄을 적용한다:

$$\text{lr}(\text{iter}) = \text{lr}_{\text{init}} \cdot \exp\left(\frac{\text{iter}}{\text{max\_iter}} \cdot \ln\frac{\text{lr}_{\text{final}}}{\text{lr}_{\text{init}}}\right)$$

이는 초기에 빠르게 대략적인 위치를 잡고, 후반에 미세 조정하기 위함이다.

---

## 8. NeRF와의 비교 정리

| 구분 | NeRF | 3D Gaussian Splatting |
|------|------|----------------------|
| **장면 표현** | Implicit (MLP) | Explicit (가우시안 집합) |
| **렌더링 방식** | Ray marching | 래스터화 (Splatting) |
| **학습 시간** | 수 시간 ~ 수 일 | 수십 분 |
| **렌더링 속도** | ~1 FPS | 100+ FPS (1080p) |
| **편집 가능성** | 어려움 | 포인트 단위 조작 가능 |
| **메모리** | MLP 파라미터 (~수 MB) | 가우시안 파라미터 (~수백 MB) |
| **기하학 추출** | Marching cubes 필요 | 직접 포인트 클라우드로 사용 가능 |

---

## 9. 한계와 후속 연구 방향

**논문이 인정하는 한계:**
- 학습 뷰에서 충분히 관측되지 않은 영역에서 아티팩트 발생
- 가우시안 수가 급격히 증가할 수 있어 메모리 소비가 큼
- 큰 가우시안이 가까이에서 관찰될 때 popping artifact 발생

**이후 연구들이 해결한 문제들:**
- **동적 장면**: Dynamic 3D Gaussians, 4D Gaussian Splatting
- **압축**: Compact 3DGS, 양자화 및 가지치기 기법
- **정규화**: 2D Gaussian Splatting (표면 정렬), SuGaR (메시 추출)
- **일반화**: pixelSplat, MVSplat (feed-forward 방식)
- **물리 기반**: PhysGaussian, GaussianGrasping

---

## 핵심 통찰

이 논문의 근본적 기여는 **표현(representation)의 선택이 렌더링 알고리즘을 결정하고, 렌더링 알고리즘이 성능 상한을 결정한다**는 점을 명확히 보여준 것이다. NeRF가 implicit representation → ray marching의 경로를 택한 반면, 3DGS는 explicit primitives → rasterization의 경로를 택했다. 후자의 선택이 GPU 하드웨어의 병렬 처리 구조에 훨씬 적합했고, 이것이 100배 이상의 렌더링 속도 차이로 이어졌다.

동시에, 미분 가능한 래스터라이저를 통해 모든 가우시안 파라미터에 대한 기울기를 정확히 계산할 수 있으므로, 표현력의 손실 없이 최적화가 가능했다. 이는 "explicit이면 학습이 어렵다"는 기존의 통념을 깨뜨린 결과이다.
