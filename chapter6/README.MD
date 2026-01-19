# **랭그래프로 설계하는 RAG 파이프라인**  
랭그래프(LangGraph)는 LLM 기반 에이전트 시스템을 쉽고 효율적으로 개발할 수 있도록 지원하는 라이브러리다. 여러 개의 에이전트로 구성된 LLM 애플리케이션과 
RAG 파이프라인에서 자주 사용되는 순환(loop)과 분기(branch)를 포함한 복잡한 워크플로우를 간편하게 구현할 수 있도록 다양한 기능을 제공한다.  
  
다음과 같은 예를 떠올려 보자  
- LLM을 사용하여 생성된 답변이 충분한지 혹은 답변을 재생성할지 결정한다.  
- LLM을 사용하여 어떤 툴(Tool)을 호출할지 결정한다.  
  
이처럼 순환 구조를 갖는 LLM 파이프라인 안에는 여러 분기점이 존재할 수 있으며 이러한 분기점이 많아질수록 시스템 구현은 더욱 복잡해진다. 랭그래프는 
그래프 구조를 활용해 이를 간단하게 구현할 수 있도록 도우며 다양한 제어 방식을 제공해 애플리케이션의 기능과 유연성을 한층 강화한다.  
  
# **랭그래프의 구성요소**  
먼저 그래프의 기본 개념을 짚어본다. 그래프는 객체들 간의 관계를 나타낼 때 사용하는 데이터 구조로 노드와 에지로 이루어져 있다.  
  
- 노드(Node, 정점): 그래프에서 개별 객체를 나타내는 요소이다. 예를 들어 소셜 네트워크를 그래프로 표현한다면 각 사용자가 하나의 노드가 된다.  
- 에지(Edge, 간선): 드 노드를 연결하는 요소로 노드 간의 관계 또는 경로를 나타낸다. 소셜 네트워크를 그래프로 표현한다면 사람들 사이의 친구 관계가 
에지가 된다.  
  
랭그래프에서는 이러한 그래프 구조를 이용해 다양한 워크플로우를 구현한다. 기본 동작 방식은 노드가 작업을 완료하면 하나 이상의 에지를 통해 다른 노드에게 
메시지를 보내고 메시지를 받은 노드는 자신의 기능을 실행한 후 다음 노드로 메시지를 다시 전달하는 과정을 반복하는 방식이다.  
  
# **그래프**  
랭그래프의 그래프는 구글의 프리겔(Pregel)에서 영감을 받은 슈퍼스텝(super-steps) 방식으로 동작한다. 슈퍼스텝은 그래프 처리 과정의 한 단계로 각 노드가 
병렬로 동시에 작업을 수행하는 단위다. 병렬로 작업을 수행한다는 것은 한 노드가 일을 끝낸 후에 다른 노드가 일을 시작하는 것이 아니라 여러 노드가 동시에 
자신의 일을 하는 모습을 뜻한다. 이렇게 동시에 실행되는 노드는 동일한 슈퍼스텝에 속하며 순차적으로 실행되는 노드는 별도의 슈퍼스텝에 속한다.  
  
노드는 하나 이상의 입력 에지에서 새로운 메시지(상태)를 수신할 때 활성화된다. 활성화된 노드는 자신의 기능을 실행하고 처리 결과를 다른 노드로 전달한다. 
각 슈퍼스텝이 끝날 때 입력 메시지가 없는 노드는 자신을 비활성화료 표시하여 완료된 상태로 표시한다. 모든 노드가 비활성화되고 더 이상 메시지가 
전송 중이지 않은 상태가 되면 그래프 실행이 종료된다.  
  
![img.png](image/img.png)  
  
랭그래프에서는 두 가지 유형의 그래프 클래스를 사용할 수 있다.  
  
- 상태 그래프(StateGraph): 일반적으로 사용하는 그래프 클래스다. 사용자가 정의하는 상태를 매개변수로 활용하며 다양한 워크플로우와 상태 관리를 
지원한다.  
- 메시지 그래프(MessageGraph): 오직 메시지 목록만으로 이루어지는 특별한 유형의 그래프 클래스다. 주로 챗봇과 같은 대화형 시스템에서 사용되며 메시지 
흐름을 단순화하여 관리하기에 적합하다.  
  
# **상태**  
그래프를 정의할 때 가장 먼저 해야 할 일은 그래프의 상태(state)를 정의하는 것이다. 상태란 애플리케이션 내에서 메시지로 주고받는 변수들의 집합이다. 
파이썬의 모든 타입으로 정의할 수 있지만 대체로 TypedDict나 Pydantic의 BaseModel 타입으로 선언한다. 상태는 그래프 내 모든 노드와 에지의 입력으로 
사용되며 각 노드의 상태를 업데이트할 수 있다.  
  
```
from typing import TypedDict

class State(TypedDict):
    count: int
    messages: list[str]  
```  
  
이 예시는 count와 messages라는 두 개의 필드를 갖는 상태 클래스다. 이 상태는 그래프 내 여러 노드에서 공유되며 지속적으로 업데이트된다.  
  
예를 들어 처음 노드에서 {"count": 1, "messages": ["hi"]}를 입력하고 다음 노드에서 {"count": 2}를 입력한다면 상태는 {"count": 2, "messages": ["hi"]}의 
형태가 된다. 그다음 노드에서 {"messages": ["bye"]}를 입력한다면 최종 상태는 {"count": 2, "messages": ["bye"]}의 형태가 된다.  
  
![img.png](image/img2.png)  
  
리듀서(reducer)를 사용한다면 기존 상태에 새로운 업데이트를 결합하여 새로운 상태를 생성하는것도 가능하다. 다음 예시와 같이 Annotated 타입으로 
리듀서 함수를 정의한다면 messages 변수는 리듀서 함수를 통해 업데이트된다.  
  
```
from typing import TypedDict, Annotated
from operator import add 

class State(TypedDict):
    count: int
    messages: Annotated[list[str], add]
```  
  
이 예시에서는 messages 필드에 add 리듀서가 지정되어 있어 새로운 메시지가 추가될 때 기존 리스트와 병합된다.  
  
예를 들어 처음 노드에서 {"count": 1, "messages": ["hi"]}를 입력하고 다음 노드에서 {"count": 2}를 입력한다면 상태는 {"count": 2, "messages": ["hi"]}의 
형태가 된다. 그다음 노드에서 {"messages": ["bye"]}를 입력한다면 최종 상태는 messages에 operator.add를 실행하여 {"count": 2, "messages": ["hi", "bye"]}의 형태가 된다.  
  
![img.png](image/img3.png)  
  
# **노드**  
그래프에서 노드(node)는 실제 작업을 수행하는 실행 단위이다. 에이전트의 로직을 담은 파이썬 함수가 곧 노드가 되며 그래프에서 각 노드는 상태를 입력으로 
받아 동작하고 그 결과로 상태값을 업데이트하여 반환한다. 노드는 정상적으로 실행될 수도 있고 실패할 수도 있다. 한마디로 노드는 그래프 내에서 실질적인 
작업을 처리하는 핵심 구성 요소다.  
  
노드는 첫 번째 인자로 상태값(state)을 받으며 두 번째 인자로 설정값(config)을 받는다. 이렇게 생성한 노드는 add_node() 메서드를 통해 그래프에 추가할 
수 있다.  
  
```
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph

# 상태 그래프 선언
builder = StateGraph(dict)

# 노드로 사용할 함수 정의
def my_node(state: dict, config: RunnableConfig):
    print("In node: ", config["configurable"]["user_id"])
    
    return {"result": f"Hello, {state['input']}!"}

def my_other_node(state: dict):
    return state # 상태를 그대로 반환
    
# 노드를 그래프에 추가
builder.add_node("my_node", my_node)
builder.add_node("other_node", my_other_node)
```  
  
앞의 코드에서 my_node는 입력된 상태값을 변경하기 않고 그대로 반환하는 간단한 노드다.  
  
# **START 노드**  
START 노드는 그래프 실행의 시작점을 나타내는 특별한 노드다. 사용자 입력을 처음 받아 그래프로 전달하며 그래프 내에서 첫 번재로 실행될 노드를 지정할 
때 사용한다.  
  
```
from langgraph.graph import START

graph.add_edge(START, "node_a")
```  
  
앞의 코드에서는 START 노드가 실행된 후 node_a로 상태가 전달된다.  
  
# **END 노드**  
END 노드는 그래프 실행이 완료되었음을 나타내는 종료 노드다. 특정 노드의 작업이 끝난 후 더 이상 처리할 작업이 없을 경우 END 노드로 연결하여 그래프 
실행을 종료한다.  
  
```
from langgraph.graph import END

graph.add_edge("node_a", END)
```  
  
이 코드는 node_a가 실행된 후 그래프가 종료되도록 설정한 예이다.  
  
# **에지**  
에지(edge)는 그래프 내에서 노드가 작업을 수행한 후 다음에 어떤 동작을 이어갈지 결정하는 흐름 제어 요소다. 파이썬 함수나 고정된 연결을 통해 다음 
실행 노드를 지정하며 조건에 따라 분기하거나 종료를 지시할 수도 있다. 한마리도 에지는 그래프 실행 흐름을 설계하는 핵심 구성요소다.  
  
하나의 노드는 여러 개의 에지를 가질 수 있으며 각각의 에지를 통해 다양한 경로로 분기하거나 순차적으로 작업을 이어갈 수 있다.  
  
# **일반 에지**  
가장 기본적인 형태로 한 노드에서 다음 노드로 직접 이동할 때 사용한다.  
  
```
graph.add_edge("node_a", "node_b")
```  
  
이 코드는 node_a의 작업이 끝난 후 node_b가 실행되도록 지정한다.  
  
# **조건부 에지**  
특정 조건에 따라 다른 노드로 분기하거나 플로우를 종료할 때 사용한다. 조건을 판단하는 함수가 필요하며 그 결과에 따라 다음 실행 노드를 선택한다.  
  
```
graph.add_conditional_edges("node_a", routing_function,
                            {True: "node_b", False: "node_c"})
```
  
이 코드에서는 routing_function이 node_a 다음으로 어떤 노드를 사용할지 결정하는 함수가 되며 해당 함수의 반환값이 True이면 node_b가, False면 
node_c가 다음 실행 노드가 된다.  
  
# **진입 지점**  
진입 지정(entry point)은 그래프가 시작될 때 처음 실행할 노드를 명시한다. 주로 START라는 가상의 노드를 사용하여 그래프의 첫 실행 지점을 설정한다.  
  
```
from langgraph.graph import START

graph.add_edge(START, "node_a")
```  
  
이 코드는 START 노드에서 첫 번째로 실행할 노드로 에지를 추가하는 add_edg() 메서드를 사용하여 그래프의 진입 지점을 지정한다.  
  
# **조건부 진입 지점**  
사용자의 입력이나 외부 조건에 따라 그래프 실행의 첫 번째 노드를 동적으로 결정할 수 있다. 진입 지점 역시 조건부로 설정하여 다양한 초기 상태에 대응한다.  
  
```
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function,
                            {True: "node_b", False: "node_c"})
```  
  
add_conditional_edge() 메서드를 사용하여 가상의 START 노드와 라우팅 함수를 입력받는다. 세 번째 인자로 라우팅 함수의 반환값에 해당하는 노드의 
매핑 정보를 제공하여 조건부로 진입 지점 노드를 선택할 수 있다. 앞의 코드에서는 routing_function이 진입 지점으로 어떤 노드를 사용할지 결정하는 함수가 
되며 해당 함수의 반환값이 True이면 node_b가, False이면 node_c가 첫 실행 노드가 된다.  
  
