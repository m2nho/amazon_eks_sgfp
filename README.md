# SGFP (Security Groups for Pods) 실습 요약

Amazon EKS에서 Pod별 보안 그룹을 설정할 수 있는 기능인 SGFP(Security Groups for Pods)를 실습한 내용을 정리하였습니다. SGFP를 사용하면 Pod 단위의 네트워크 보안 제어가 가능하며, 세밀한 접근 제어 및 최소 권한 원칙을 적용할 수 있습니다.

---

## ✅ SGFP 미적용 구조와 한계

![SGFP 미적용](/images/page1.png)

- 모든 Pod가 노드의 ENI와 보안 그룹을 공유함
- 모든 Pod는 RDS와 같은 자원에 접근 가능 → 과도한 권한 발생
- 세밀한 제어 불가

---

## ✅ SGFP 적용 구조와 장점

![SGFP 적용](/images/page2.png)

- Pod마다 별도의 보안 그룹 적용 가능
- 특정 Pod만 RDS 접근 허용 가능
- 최소 권한 원칙 적용
- 세밀한 제어 가능


---

## ✅ SGFP를 위한 설정 요소

### Trunk ENI와 Branch ENI

![SGFP ENI 설정](/images/page5.png)

- Trunk ENI는 Pod별 Branch ENI를 연결할 수 있는 구조
- ENI 슬롯 제약 없이 Pod별 개별 ENI 생성 가능

---

### ENABLE_POD_ENI=true

![ENABLE_POD_ENI 설정](/images/page6.png)

```
kubectl set env daemonset aws-node -n kube-system ENABLE_POD_ENI=true
```

- aws-node DaemonSet에 환경변수 추가
- ```kubectl get cninode -A``` 로 적용 상태 확인

---

## ✅ SGFP를 위한 IAM 권한 설정

![SGFP 권한](/images/page7.png)

```
# 클러스터에 연결된 Role ARN 가져오기
cluster_role=$(aws eks describe-cluster --name my-eks-cluster --query cluster.roleArn --output text | cut -d / -f 2)

# 정책 연결
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
  --role-name $cluster_role
```

- ENI 생성, 수정, 삭제 등의 권한 필요

---

## ✅ SGFP 적용 조건

![SGFP 조건](/images/page8.png)

- **지원 인스턴스 타입**: m5, c5, r5, m6g, c6g, r6g 등 (`t` 패밀리는 미지원)
- **지원 CNI 버전**:
  - EC2 노드: 1.16.0 이상
  - Fargate: 1.7.7 이상
- Windows 노드는 미지원

---

## ✅ SecurityGroupPolicy 설정

![SecurityGroupPolicy 생성](/images/page9.png)

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

- Pod Label 기반으로 Security Group을 매핑
- SG는 미리 생성해두어야 함

---

## ✅ Pod 생성 시 ENI 및 IP 변화 확인

![ENI 변화 및 IP 할당](/images/page10.png)

- Pod 생성 시 Branch ENI에 개별 IP 할당
- ```kubectl get po -o wide``` 및 ENI IP 리스트로 확인 가능
- Pod가 Node의 보조 IP로 매핑되어 RDS 또는 EC2와 직접 통신 가능

---

## 💡 SGFP 사용 이유 및 요약

| 항목 | 설명 |
|------|------|
| 목적 | Pod 별 세밀한 네트워크 보안 제어 |
| 장점 | 최소 권한 원칙, 특정 자원 접근 통제, 보안 그룹 유연한 설정 |
| 주요 구성 요소 | Trunk ENI, Branch ENI, ENABLE_POD_ENI, IAM 권한, SecurityGroupPolicy |
| 필수 조건 | EC2 인스턴스 타입 제한, CNI 플러그인 버전 조건 충족 |
| 적용 효과 | 특정 Pod만 EC2, RDS 접근 가능하게 구성 가능 |

---
