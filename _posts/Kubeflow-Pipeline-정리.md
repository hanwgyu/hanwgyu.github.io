##  파이프라인 내용 정리

### 파이프라인 SDK 패키지

- kfp.compiler : 파이프라인 컴포넌트를 빌드
   - .Compiler.compile : 파이썬 DSL 코드를 파이프라인 YAML 파일로 변환
- kfp.components :  파이프라인 컴포넌트 관련 클래스, 모듈들
    - .func_to_conatiner_op : 파이썬 함수를 파이프라인 컴포넌트로 변환
   - .load_component_from_file : 파일을 파이프라인 컴포넌트로 변환
   - .load_component_from_url: URL에서 파이프라인 컴포넌트로 변환
- kfp.dsl : 파이프라인을 파이선 코드로 작성할 때 사용하는 클래스와 메소드, 모듈들의 패키지
   - .ContainerOp : 컨테이너 이미지를 다룸
   - .PipelineParam : 파이프라인 간 전달되는 파라미터
   - .ResourceOp : 쿠버네티스 리소스를 다룸
   - .VolumeOp : 쿠버네티스 PVC를 생성
   - .VolumeSnapshotOp : 볼륨의 스냅샷을 생성
   - .PipelineVolume : 기존의 PVC를 사용하거나, 파이프라인끼리 데이터 공유용으로 사용되는 볼륨을 설정. ContainerOp에서 pvolume 옵션으로 마운트 가능
- kfp.Client : 파이프라인 API와 통신하는 파이썬 클라이언트 라이브러리 패키지
   - .create_experiment : experiment 생성
   - .run_pipeline : run 생성
   - .create_run_from_pipeline_func : python 함수를 파이프라인 run으로 생성. 노트북에서 실행시 Experiment, Run에 대한 링크를 제공. 파이썬파일을 pipeline package로 컴파일한후 create_run_from_pipeline_package를 실행하게 구현되어 있음
   - .create_run_from_pipeline_package : 컴파일된 pipeline package를 run으로 생성
   - .upload_pipeline : 컴파일된 pipeline package를 kubeflow pipeline cluster로 업로드. 노트북일 경우 링크제공.

--- 
### 파이프라인 컴퍼넌트 생성 방법

#### 1. Creating components from existing application code 
도커 이미지를 그대로 사용해 파이프라인 생성

필요한 테스크를 수행하는 애플리케이션이 도커 이미지로 패키징되어 단독으로 실행할 수 있어야함. 해당 도커 이미지를 넣어서 작성.

컨테이너 이름과 실행 명령어, 인자값, 사이드카 컨테이너, 아웃풋 파일 저장 경로, 볼륨 경로 등을 넣어줌.

dsl.ContainerOp 함수에 도커이미지, 실행 커맨드, argument, PVC, GPU 사용정보 등을 넣어서 파이프라인 컴포넌트 생성. 

``` python
# Write a component function using the Kubeflow Pipelines DSL to define your pipeline’s interactions with the component’s Docker container
@kfp.dsl.component
def my_component(my_param):
  ...
  return kfp.dsl.ContainerOp(
    name='My component name',
    image='gcr.io/path/to/container/image'
  )
```

#### 2. Creating components within your application code
파이썬 코드를 기반으로 도커 이미지 생성.

```python
# convert your Python function into a pipeline component
@kfp.dsl.python_component(
  name='My awesome component',
  description='Come and play',
)
def my_python_func(a: str, b: str) -> str:
  ...


# create a container image for the component
my_op = compiler.build_python_component(
  component_func=my_python_func,
  staging_gcs_path=OUTPUT_DIR,
  target_image=TARGET_IMAGE)
```

#### 3. Creating lightweight components
도커 빌드 없이 파이썬 함수를 그대로 이용.
파이썬 함수 하나로 만들어야하며, 함수 선언 외부에 어떤 코드도 선언되어서는 안되며, import도 메인 함수 안에 있어야하며, 다른 함수를 사용하려면 메인 함수 안에서 선언되어야 한다.
함수를 만든 후, kfp.components.func_to_container_op(func) (kfp.components.create_component_from_func 와 거의 동일) 모듈을 사용해 컴포넌트로 변환한다. (베이스 이미지를 넣어서 함수에 필요한 패키지를 가져올 수 있다)
매개 변수에 유형 힌트가 있어야함. int, float, bool 외엔 모두 문자열로 인식함.
파일로 생성해낼수도 있다.

```  python
# Write your code in a Python function
def my_python_func(a: str, b: str) -> str:
  ...


# convert your Python function into a pipeline component
# you can write the component to a file that you can share or use in another pipeline
my_op = kfp.components.func_to_container_op(my_python_func, output_component_file='my-op.component')

# If you stored your lightweight component in a file as described in the previous step, use kfp.components.load_component_from_file to load the component:
# my_op = kfp.components.load_component_from_file('my-op.component')
```


#### 4. Using prebuilt, reusable components in your pipeline

Pipeline component YAML파일을 사용
``` python
# load the component from yaml url
my_op = kfp.components.load_component_from_url('https://path/to/component.yaml')
```

`kfp.components.create_component_from_func` 함수를 이용해 yaml 파일을 생성해낼 수 있다.( 3번의 `kfp.components.func_to_container_op` 함수를 통해 생성해내는 것과 동일한지는 모르겠음)
```python
my_op = kfp.components.create_component_from_func(my_python_func, output_component_file='component.yaml')
```

--- 
### 생성된 컴포넌트들을 이용해 파이프라인 함수를 작성
컴포넌트들을 불러와서 after 함수로 실행 순서를 정하고 파이프라인 컴포넌트들을 연결해줄 수 있다.
``` python
# Write a pipeline function using the Kubeflow Pipelines DSL
@kfp.dsl.pipeline(
  name='My pipeline',
  description='My machine learning pipeline'
)
def my_pipeline(param_1: PipelineParam, param_2: PipelineParam):
  my_step1 = my_op(a='a', b='b')
  
  my_step2 = dsl.VolumeOp(...)
  my_step3 = dsl.ContainerOp(...)
  
  # my_step1 -> my_step2 -> my_step3
  my_step2.after(my_step1)
  my_step3.after(my_step2)
```

특정 컴포넌트의 실행 결과 값은 볼륨을 마운트해서 파일에 저장하고, 다음 컴포넌트에서 동일한 볼륨을 마운트하여 파일을 읽어들여서 사용해야함

---
### 파이프라인 함수 공유
``` python
kfp.compiler.Compiler().compile(my_pipeline,'my-pipeline.zip')
```
또는
``` shell
$ dsl-compile --py [path/to/python/file] --output my-pipeline.zip
```

Pipeline function을 위와 같이 compile해서 zip 파일이나 tar.gz 파일로 생성할 수 있음.
생성된 pipeline zip 파일을 Kubeflow Pipelines Web UI에 업로드해서 Run을 생성할 수 있다.

---
### 파이프라인 실행
#### 1. 파이썬 코드에서 실행
run_pipeline 함수에서는 생성된 zip파일을 이용하거나 코드에서 작성한 pipeline function을 그대로 사용할 수도 있다.
``` python
client = kfp.Client()
my_experiment = client.create_experiment(name='demo')

# Run using a zip file
my_run = client.run_pipeline(my_experiment.id, 'my-pipeline', 'my-pipeline.zip')

# Run using the pipeline function
# my_run = client.run_pipeline(my-pipeline, arguments={'param_1':1, 'param_2:2})
```

#### 2. 웹 UI에서 실행
생성된 zip 파일을 불러와 Run을 생성할 수 있다.


---
### Reference
- 파이프라인 공식 가이드 https://www.kubeflow.org/docs/pipelines/sdk/sdk-overview/
- '쿠버네티스에서 머신러닝이 처음이라면! 쿠브플로우!' 책
