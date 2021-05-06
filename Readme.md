# Kubernetes 

https://kubernetes.io/docs/home/

***

## ConfigMap

컨피그맵은 Key-Value 쌍으로 기민이 아닌 데이터를 저장하는데 사용하는 API 오브젝트다. 
  - 컨피그맵은 보안 또는 암호화를 제공하지 않으므로 저장하려는 데이터가 보안되어야 할 경우 컨피그맵 대신 시크릿을 써야한다.

파드는 Volume 에서 환경 변수, 커맨드-라인 인수 또는 설정 파일로 컨피그맵을 사용할 수 있다. 

컨피그맵은 데이터 1MB를 초과할 수 없다. 

파드와 컨피그맵은 동일한 네임스페이스에 있어야 사용이 가능하다

다른 쿠버네티스 오브젝트와는 달리 컨피그맵에는 data 와 binaryData 필드가 있다. 이러한 필드는 키-값 싼을 값으로 허용한다. 예제를 통해 이 필드를 어떻게 사용하는지 보자.

##### ConfigMap: Key And File 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 속성과 비슷한 키; 각 키는 간단한 값으로 매핑됨
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 파일과 비슷한 키
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

컨피그맵을 사용해서 파드 내부 컨테이너 환경을 구성하는 방법은 4가지가 있다. 
- 컨테이너 커맨드와 인수를 이용해서
- 컨테이너 환경 변수를 통해서
- 애플리케이션이 읽을 수 있도록 Volume에 추가하는 방법
- 쿠버네티스 API를 이용해서 컨피그맵을 읽는 파드 내에서 실행할 코드 작성

처음 세가지 방법은 Kubelet 이 컨테이너를 시작할 때 컨피그맵의 데이터를 사용한다. 네번째 방법은 컨피그맵과 데이터를 읽기 위해 코드를 작성해야한다. 그치만 쿠버네티스 API 를 직접 사용하기 떄문에 컨피그맵의 변경 사항에 대해 구독하고 있고 이 업데이트를 받을 수 있다.  

다음은 위의 `game-demo`  의 값을 사용해서 파드를 구성하는 예시다. 

##### ConfigMap: Pod Example
```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: configmap-demo-pod
  spec:
    containers:
      - name: demo
        image: alpine
        command: ["sleep", "3600"]
        env:
          # 환경 변수 정의
          - name: PLAYER_INITIAL_LIVES # 참고로 여기서는 컨피그맵의 키 이름과
                                       # 대소문자가 다르다.
            valueFrom:
              configMapKeyRef:
                name: game-demo           # 이 값의 컨피그맵.
                key: player_initial_lives # 가져올 키.
          - name: UI_PROPERTIES_FILE_NAME
            valueFrom:
              configMapKeyRef:
                name: game-demo
                key: ui_properties_file_name
        volumeMounts:
        - name: config
          mountPath: "/config"
          readOnly: true
    volumes:
      # 파드 레벨에서 볼륨을 설정한 다음, 해당 파드 내의 컨테이너에 마운트한다.
      - name: config
        configMap:
          # 마운트하려는 컨피그맵의 이름을 제공한다.
          name: game-demo
          # 컨피그맵에서 파일로 생성할 키 배열
          items:
          - key: "game.properties"
            path: "game.properties"
          - key: "user-interface.properties"
            path: "user-interface.properties"
```

파드에서 컨피그맵을 파일로 사용할려면 다음과 같이 하면 된다. 

##### ConfigMap: Pod Volume
```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mypod
      image: redis
      volumeMounts:
      - name: foo
        mountPath: "/etc/foo"
        readOnly: true
    volumes:
    - name: foo
      configmap:
        name: myconfigmap
```

마운트된 컨피그맵은 자동으로 업데이트 되지만 환경 변수로 사용하는 컨피그맵은 자동으로 업데이트 되지 않는다. 그러므로 파드를 다시 시작해야한다. 

컨피그맵의 업데이트를 좀 더 살펴보자면 Kubelet은 모든 주기적인 동기화에서 컨피그맵이 최신 상태인지 확인한다. 이떄 로컬캐시를 사용한다. 그러므로 업데이트 되는 시간까지는 kubelte의 동기화 기간 + 캐시 전파 지연이 될 수 있다. 

컨피그맵은 변경할 수 없도록 할 수 있다. 변경을 막아 놓으므로 애플리케이션 중단을 일으킬 원인을 사전에 막아놓는게 가능하다. 그리고 변경을 막아 놓았으니까 컨피그맵이 변경되었는지 감지하는 동기화 작업을 하지 않아도 되므로 성능상에 이점이 있다. 

다음 예는 Immutable ConfigMap의 예이다.

##### ConfigMap: Immutable ConfigMap 
```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    ...
  data:
    ...
  immutable: true
```

*** 
 
