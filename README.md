<img src="https://capsule-render.vercel.app/api?type=waving&color=00C3FF&height=150&section=header" width="1000" />

<div align="center">
<h1 style="font-size: 36px;"> 💝 Kubernetes PV & PVC 설계 및 검증 </h1>
</div>
<p align="center">
  <i>컨테이너 환경에서 데이터를 안전하고 지속적으로 관리하기 위한 스토리지 설계 실습</i>
</p>

</br>

## 🎯 프로젝트 개요

- 이 프로젝트는 Kubernetes 환경에서 **PersistentVolume(PV)** 와 **PersistentVolumeClaim(PVC)** 를 활용해,  
**Pod 재시작 혹은 삭제 이후에도 데이터가 유지되는지**를 실습하고 확인하는 데 목적이 있습니다.

- 데이터가 **컨테이너 내부가 아닌 외부의 물리 디스크에 안전하게 저장되는 방식**을 이해하고,  
PV/PVC가 어떻게 작동하는지에 대한 개념을 실습을 통해 명확히 합니다.

<br>

## 🛠 실습 구성 흐름

1. `busybox` 컨테이너를 활용한 Pod 생성
2. Pod 내부 `/data` 디렉토리에 텍스트 파일 작성
3. 해당 디렉토리를 PVC로 마운트하여 PV에 저장
4. Pod를 삭제 후 재생성하여 데이터가 유지되는지 검증

<br>


## ✏️ 주요 개념

| 구성 요소 | 설명 |
| -------- | ---- |
| 📦 **PV (PersistentVolume)** | 클러스터에 존재하는 실제 저장 장치 (Node의 디스크, NFS 등) |
| 📑 **PVC (PersistentVolumeClaim)** | Pod가 필요로 하는 저장 공간 요청 (요청서 역할) |
| 🐚 **Pod** | PVC를 통해 실제 PV에 마운트된 디렉터리를 사용하는 컨테이너 |

<br>


## 🗂️ 파일 구성

| 파일       | 설명 |
|------------|------|
| `pv.yaml`  | 실제 저장 공간(`/mnt/data`)을 사용하는 PV 생성 설정 |
| `pvc.yaml` | 500Mi 용량을 요청하는 PVC 생성 설정 |
| `pod.yaml` | BusyBox 컨테이너가 PVC를 통해 `/data`에 마운트되는 Pod 구성 파일 |


<details>
  <summary><strong>pv.yaml</strong></summary>

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: busybox-pv
  spec:
    capacity:
      storage: 1Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    hostPath:
      path: /mnt/data
  ```
</details>


<details> <summary><strong>pvc.yaml</strong></summary>

  ```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: busybox-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  ```
</details>


<details> <summary><strong>pod.yaml</strong></summary>

  ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - mountPath: /data
          name: storage
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: busybox-pvc
  ```
</details>

<br>

## ⚙️ 실습 과정

## 1️⃣ Busy Box 설치 
- Docker를 사용하여 BusyBox 이미지 다운로드 진행

```bash
  $docker pull busybox
  ```

<br>

## 2️⃣ 리소스 적용 
- 위 YAML 파일 저장 후, 해당 디렉토리에서 명령어 실행 

```bash
# pv_practice 디렉토리에 yaml 파일 저장 
# 위치로 이동하여 yaml 파일 적용 

$kubectl apply -f pv.yaml
$kubectl apply -f pvc.yaml
$kubectl apply -f pod.yaml
```

<br>

## 3️⃣ 검증

- 리소스 상태 확인

```
$kubectl get pv
$kubectl get pvc
$kubectl get pods
```
<img src="https://github.com/user-attachments/assets/6d9bd82e-414f-4747-8900-07f738c5b20e" width="600"/>

<br>

### 💨 컨테이너 진입 및 데이터 생성

```bash
# 컨테이너 진입
$kubectl exec -it busybox-pod -- sh

# `/data` 디렉터리에서 데이터 입력 후 확인
cd /data
echo "PVC I'M HERE!!!!!!!" > hello.txt
cat hello.txt
```
<img src="https://github.com/user-attachments/assets/3c6c9b9a-7251-42eb-ab92-f3c332f373f4" width="600"/>

<br>

### 💫 Pod 삭제 후 데이터 영속성 확인 

```bash
# busybox pod 삭제 
$kubectl delete pod busybox-pod

# pod.yaml 파일 적용 
$kubectl apply -f pod.yaml

# pod 진입하여 데이터 영속성 확인
$kubectl exec -it busybox-pod -- sh
$cat /data/hello.txt
```
<img src="https://github.com/user-attachments/assets/158b5ecd-eb47-4ad7-9709-f62bf093bb45" width="600"/>

<br>

✅ 또한, 실제 Node(/mnt/data)에서 데이터가 유지되는지도 확인 가능 <br>
<img src="https://github.com/user-attachments/assets/364d0acd-290d-4ed3-b54a-35b095bc0366" width="600"/>

<br>

## 🔍 회고

- `hostPath`는 로컬 환경에서만 적절하므로, NFS 또는 동적 프로비저닝(StorageClass) 으로 확장 필요
- StatefulSet 환경에서의 PVC 활용, Pod 수평 확장 시의 동작 방식도 함께 실습해보면 좋을 것 같음
- Pod 재기동이 아닌 Node 장애 상황에서도 데이터가 보존되는지에 대한 실험도 필요




