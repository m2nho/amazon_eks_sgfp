# SGFP (Security Groups for Pods) 실습 요약

EKS에서 Pod 단위로 보안 그룹을 부여할 수 있도록 하는 SGFP(Security Groups for Pods) 실습 내용을 정리한 문서입니다.

---

## 1. SGFP 미적용 시 구조

![page1](/images/page1.png)

- 모든 Pod는 노드와 동일한 보안 그룹을 공유
- 모든 Pod가 RDS에 접근 가능하여 최소 권한 원칙이 지켜지지 않음
- 과도한 권한 부여 및 세밀한 제어 불가

---

## 2. SGFP 적용 시 구조

![page2](/images/page2.png)

- Pod마다 개별 보안 그룹 지정 가능
- 특정 Pod만 RDS에 접근 가능하도록 설정 가능
- 최소 권한 원칙 적용
- 세밀한 네트워크 제어 가능

---

## 3. SGFP 조건

![page3](/images/page3.png)

- **지원 인스턴스 타입**: m5, c5, r5, m6g, c6g, r6g 등 (`t` 계열은 미지원)
- **운영체제**: Windows Node 미지원
- **CNI 버전**: EC2 Node는 1.16.0 이상, Fargate는 1.7.7 이상 필요

---

## 4. SGFP를 위한 EKS IAM 권한 설정

![page4](/images/page4.png)

```
# 클러스터에 연결된 IAM Role 이름 가져오기
cluster_role=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --query cluster.roleArn \
  --output text | cut -d / -f 2)

# IAM Role에 권한 부여
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
  --role-name $cluster_role
```

- ENI 생성, 수정, 삭제 권한이 필요
- JSON 정책 파일을 통해 세부 권한 제어 가능

---

## 5. ENABLE_POD_ENI 활성화

![page5](/images/page5.png)

```
# aws-node DaemonSet에 환경 변수 추가
kubectl set env daemonset aws-node \
  -n kube-system ENABLE_POD_ENI=true
```

- SecurityGroupsForPods 기능이 활성화되었는지 확인
```
kubectl get cninode -A
```

---

## 6. Trunk ENI와 Branch ENI 구조

![page6](/images/page6.png)

- EC2 인스턴스에 Trunk ENI를 추가하여 Branch ENI로 Pod에 ENI 할당
- Pod는 ENI Slot 수와 무관하게 독립된 네트워크 인터페이스 사용 가능

---

## 7. SecurityGroupPolicy 적용

![page7](/images/page7.png)

```
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-security-group-policy
spec:
  podSelector:
    matchLabels:
      app: my-app
  securityGroups:
    groupIds:
      - sg-08a1955830794145
```

- 특정 라벨을 가진 Pod에 특정 보안 그룹을 지정하는 리소스

---

## 8. Pod ENI 상태 확인 및 IP 매핑

![page8](/images/page8.png)

- Pod 생성 시 고유한 Branch ENI를 할당받고, IP도 각각 할당됨
- ENI 상세 정보를 통해 Pod별 ENI 및 IP 확인 가능

---

## 9. 일반 방식: EC2 접근 설정

![page9](/images/page9.png)

- EC2의 보안 그룹에서 EKS Node 보안 그룹을 인바운드로 허용하면
- 모든 Pod가 EC2에 접근 가능
- 하지만 세밀한 제어는 어려움

---

## 10. SGFP 방식: EC2 접근 설정

![page10](/images/page10.png)

- EC2 보안 그룹에서 Branch ENI에 부여된 보안 그룹만 허용 가능
- 특정 Pod만 EC2에 접근 허용
- SGFP 활성화 체크리스트:
  - EKS Cluster 권한 부여
  - Trunk ENI 생성
  - SecurityGroupPolicy 생성
  - Pod 생성 및 확인

---

## 💡 SGFP 요약

| 항목         | 설명 |
|--------------|------|
| SGFP 도입 이유 | Pod 단위 보안 그룹 지정, 최소 권한 원칙 적용, 세밀한 제어 |
| 조건 | 지원 인스턴스 타입, CNI 버전, ENABLE_POD_ENI 설정 |
| 핵심 구성 | Trunk ENI, Branch ENI, IAM Role, SecurityGroupPolicy |
| 적용 효과 | Pod 별로 EC2/RDS 등 리소스에 접근 권한을 분리 제어 가능 |
