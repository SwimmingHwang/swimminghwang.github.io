---
layout: post
title: Spring Cloud Kubernetes와 Spring Scheduler를 사용한 리더 선출 기능 구현
author: Soo-young Hwang
tags:
  - Spring-Boot
  - Kubernetes
category: Spring Boot
published: true
subtitle:
---
## 서론

현업에서 한 서버에 스케쥴러 작업이 있고 이 작업이 여러 인스턴스에서 중복 실행되는 것을 막아야 할 경우가 있습니다. 이를 위해 Spring Cloud Kubernetes 를 활용하여 리더 선출(Leader Election) 메커니즘을 활용하면쉽게 특정 작업이 단일 인스턴스에서만 실행되도록 할 수 있습니다.    

새로운 기능을 구현할 때, 동작하는 방식에 대해 이해하고 있지 않으면 문제가 발생했을 때 어떤 부분을 봐야할지 모르기 때문에 동작 방식을 완전이 이해하는게 중요하다고 생각합니다. 

그래서 이번 포스트에서는 Spring Cloud Kubernetes와 Spring Scheduler를 사용하여 리더 선출 기능을 구현하는 방법을 살펴보겠습니다.

## 리더 선출의 필요성

여러 인스턴스가 동시에 주기적인 작업을 실행하면 불필요한 중복 작업이 발생할 수 있습니다. 리더 선출은 이 문제를 해결하여 클러스터 내 하나의 인스턴스에서만 특정 작업이 실행되도록 보장합니다. Kubernetes는 이러한 리더 선출 메커니즘을 지원하며, Spring Cloud Kubernetes는 이를 쉽게 애플리케이션에 통합할 수 있도록 돕
습니다.

## 동작 방식


Spring Cloud Kubernetes 리더 선거 메커니즘은 Kubernetes `ConfigMap`을 사용하여 Spring Integration의 리더 선출 API를 구현합니다.

1. 여러 애플리케이션이 구동되고 있는 상황에서 리더십을 놓고 경쟁을 한다.
2. 리더십이 부여되면 리더 애플리케이션은 리더십을 부여받았다는 이벤트를 수신하고 스케쥴러를 시작시킨다.
3. 리더가 아닌 애플리케이션은 주기적으로 리더십을 얻으려고 시도하며, 기존의 리더가 리더십을 양보하면 제일 먼저 리더십을 얻으려고 시도한 애플리케이션이 리더가 된다. 


### 라이브러리 코드와 함께보는 리더 선출 동작 과정 


> **클래스 설명** 


- Fabric8PodReadinessWatcher:
	- 이 클래스는 주기적으로 Pod의 준비 상태를 감시하고, 준비 상태가 변경되면 Fabric8LeadershipController를 통해 리더십을 업데이트
- Fabric8LeadershipController: 
	- Kubernetes에서 리더 선출을 관리
	- Pod의 준비 상태와 ConfigMap을 기반으로 리더십 갱신 및 취소 로직을 처리
- Fabric8LeaderRecordWatcher:
	- 리더 선출 정보를 저장하는 ConfigMap을 모니터링하는 클래스입니다. ConfigMap에 변화가 있을 때 `eventReceived()` 메서드가 호출되어 리더십 갱신이 트리거

> **Flow** 

- **애플리케이션 시작**:
    - 애플리케이션이 시작될 때 `Fabric8LeadershipController` 클래스가 초기화됩니다. 이 클래스는 Kubernetes 클러스터에서 리더십을 관리하고 갱신하는 역할을 합니다.
    - `Fabric8LeaderRecordWatcher` 클래스가 초기화되면서 Kubernetes 클러스터 내 ConfigMap 리소스를 감시할 준비를 합니다.

- **ConfigMap 감시 시작 (`Fabric8LeaderRecordWatcher.start()`)**:
    - `Fabric8LeaderRecordWatcher` 클래스의 `start()` 메서드가 호출되면 `configMapWatch` 객체를 생성하여 Kubernetes 클러스터에서 지정된 네임스페이스와 이름의 ConfigMap을 감시합니다.
    - 감시가 시작되면 ConfigMap의 변화가 발생할 때마다 `eventReceived()` 메서드가 호출됩니다.

- **ConfigMap 변경 감지 및 리더십 갱신 (`Fabric8LeaderRecordWatcher.eventReceived()`)**:
    - ConfigMap에 변경이 발생하면 `eventReceived()` 메서드가 호출되어 `fabric8LeadershipController.update()` 메서드를 트리거합니다.
    - 이 호출은 ConfigMap에서 현재 리더 정보를 추출하고, 필요한 경우 리더십을 갱신하거나 새로운 리더를 선출하도록 합니다.

- **리더십 갱신 및 취득 (`Fabric8LeadershipController.update()`)**:
    - `Fabric8LeadershipController` 클래스의 `update()` 메서드가 호출되면, 현재 ConfigMap의 상태를 확인하고 리더십을 취득할 수 있는지 판단합니다.
    - 현재 리더의 상태가 준비되지 않거나 다른 Pod가 리더로 설정되어 있다면, `revoke()` 메서드를 호출하여 현재 리더십을 취소하고 새로운 리더를 선출합니다.
    - 준비 상태인 Pod가 현재 후보(`Candidate`)와 동일하다면, 리더십을 취득하거나 갱신합니다.

- **Pod 준비 상태 확인 (`Fabric8LeadershipController.isPodReady()`)**:
    - `Fabric8LeadershipController`는 `isPodReady()` 메서드를 사용하여 현재 Pod가 준비 상태인지 확인합니다.
    - `Fabric8PodReadinessWatcher`는 Pod의 준비 상태가 변경될 때 이를 감지하여 리더십 갱신을 트리거하는 역할을 합니다.
	    -  Kubernetes API Server에 요청을 보내 Pod의 상태를 확인

- **리더십 취득 (`Fabric8LeadershipController.acquire()`)**:
    - 리더십을 취득할 수 있는 경우, `acquire()` 메서드는 ConfigMap에 새로운 리더 정보를 작성하거나 기존 리더 정보를 갱신합니다.
    - 리더십이 성공적으로 취득되면 `handleLeaderChange()` 메서드가 호출되어 리더십 변경 이벤트를 처리합니다.

- **리더십 취소 (`Fabric8LeadershipController.revoke()`)**:
    - Pod가 더 이상 리더가 될 수 없거나 준비 상태가 아니게 되면 `revoke()` 메서드가 호출되어 리더십이 취소됩니다.
    - ConfigMap에서 해당 리더 정보를 제거하고, `handleLeaderChange()` 메서드가 호출되어 리더십 변경 이벤트가 발생합니다.


### 1. Pod 1 리더로 선출


![https://swimminghwang.github.io/assets/images/Pastedimage20241113070616.png](https://swimminghwang.github.io/assets/images/Pastedimage20241113070616.png)


- **EKS Cluster**에 여러 Pod가 존재하며, 각 Pod는 리더 선출을 위해 `ConfigMap`을 감시합니다.
- **Pod 1**는 `Fabric8LeadershipController`, `Fabric8LeaderRecordWatcher`, `Fabric8PodReadinessWatcher`를 통해 리더십을 관리하고 상태를 모니터링합니다.
- **Pod 2**와 **Pod 3**도 `LeaderRecordWatcher`를 통해 `ConfigMap`을 지속적으로 감시하여 리더 선출 상태를 확인합니다.
- 리더십을 획득하거나 취소할 필요가 있을 때 **Pod 1**의 `Fabric8LeadershipController`가 **ConfigMap**을 갱신합니다.
- **Pod 1**이 리더가 될 경우, 다른 Pod는 **ConfigMap**을 감시하여 리더의 상태를 반영합니다.


### 2. Pod 2가 이벤트를 받아 리더가 되는 경우 

![https://swimminghwang.github.io/assets/images/Pastedimage20241113070933.png](https://swimminghwang.github.io/assets/images/Pastedimage20241113070933.png)


- **Pod 2**가 EKS 클러스터 내에서 `ConfigMap`의 변경을 감시하고, 리더 선출 이벤트를 수신하여 리더십 상태를 갱신합니다.
- `Fabric8PodReadinessWatcher`는 Kubernetes API에 Pod의 준비 상태를 확인 요청을 보냅니다.
- **Pod 2**는 준비 상태가 적절할 때 리더십을 획득하여 새로운 리더로서 역할을 수행합니다.

### 동작 요약 

recordwatcher 가 configmap watch 하고 있고 변경 이벤트를 감지 (eventReceived) 
-> leader controller update 함수 호출 
->  leaderController 에서 configmap lock 걸고 configmap에 있는 `leader.id.`로 시작하는 리더 role 데이터 확인 
경우1) **리더가 있고 그 리더 파드가 ready면** 리더를 유지하면서, 이전 리더가 나 자신(파드)이면 leader context 에 revoke 이벤트를 발행 -> 스케쥴러 stop
경우2) **새로운 리더가 나 자신이면** grant 이벤트 발행 -> 스케쥴 start 
경우3) **다른 파드가 리더면** acquire 호출해서 리더십 획득하려고 계속 시도 (내 파드가 ready고 configmap 데이터에 리더 정보가 없으면 나를 리더로 설정 -> 되풀이)

---
## 프로젝트 설정 

#### 1. 의존성 추가 

```
implementation ("org.springframework.cloud:spring-cloud-kubernetes-fabric8-leader:3.1.3")  
implementation("org.springframework.boot:spring-boot-starter-actuator")
```

- fabric8: Kubernetes 리소스를 제어할 수 있는 기능을 자바 기반으로 쉽게 사용할 수 있게 만든 라이브러리
	- Kubernetes의 다양한 리소스(예: Pod, Service, Deployment, ConfigMap 등)를 생성, 업데이트, 삭제 및 조회할 수 있습니다.
	- 기본적인 CRUD 작업뿐만 아니라 롤링 업데이트, 리소스 감시(watch), 로그 스트리밍, 리더 선출 등 다양한 고급 기능을 지원


![https://swimminghwang.github.io/assets/Pastedimage20241113062204.png](https://swimminghwang.github.io/assets/Pastedimage20241113062204.png)


#### 2. 애플리케이션 설정 


```yaml
spring:
  cloud:
	kubernetes:
	  leader:
		role: plan-scheduler
		config-map-name: test-configmap
```


#### 3. 리더 전용 작업 구현

```kotlin
@RestController  
@RequestMapping("/leader")  
public class LeaderController (
	private val managerContext: ManagerContext,
) {
	/**  
	* Handle a notification that this instance has become a leader.  
	* @param event on granted event  
	*/
	@EventListener  
	fun handleEvent(event: OnGrantedEvent) {  
	    println(String.format("'%s' leadership granted", event.role))  
	    this.context = event.context  
	    managerContext.setContext(this.context)  
	    startScheduler() // 리더가 될 때 스케줄러 시작
	}  
	  
	/**  
	 * Handle a notification that this instance's leadership has been revoked. 
	 * @param event on revoked event  
	 */@EventListener  
	fun handleEvent(event: OnRevokedEvent) {  
	    println(String.format("'%s' leadership revoked", event.role))  
	    this.context = null  
	    managerContext.setContext(null)  
	    stopScheduler()
	}

	// TODO: startScheduler
	// TODO: stopScheduler
	
	/**  
	* Return a bool type data whether this instance is a leader or not.  
	* @return isLeader data  
	*/
	private fun isLeader(): Boolean {  
	    return this.context != null
	}
}
```


#### 4. Kubernetes 리소스 구성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: {{ .Values.env }}
  name: role-{{ .Values.env }}-{{ .Chart.Name }}
  labels:
    app: role-{{ .Values.env }}-{{ .Chart.Name }}
    group: org.springframework.cloud
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - watch
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - watch
      - get
      - update
```


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: role-binding-{{ .Values.env }}-{{ .Chart.Name }}
    group: org.springframework.cloud
  namespace: {{ .Values.env }}
  name: rolebinding-{{ .Values.env }}-{{ .Chart.Name }}
roleRef:
  apiGroup: ""
  kind: Role
  name: role-{{ .Values.env }}-{{ .Chart.Name }}
subjects:
  - kind: ServiceAccount
    name: sa-{{ .Values.env }}
    apiGroup: ""
```
#### 5. Leader 확인하기 

configmap 데이터에서 확인할 수 있습니다. 

![https://swimminghwang.github.io/assets/images/Pastedimage20241113221303.png](https://swimminghwang.github.io/assets/images/Pastedimage20241113221303.png)


## 결론

직접 리더 데이터를 redis에 관리하여 작업하는 방법도 있지만 Kubernetes 기반이라면 Spring Cloud Kubernetes의 기본 리더 선출 기능을 활용하여 안전하고 간편하게 리더 기반 작업을 수행할 수 있습니다. 

이 글이 Spring Cloud Kubernetes를 활용한 리더 선출 구현에 도움이 되었기를 바랍니다. 앞으로도 더욱 안정적인 분산 시스템 구축을 위한 다양한 기술과 방법론을 다룰 예정이니, 많은 관심 부탁드립니다! 