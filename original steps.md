### 1. `vector-search-lab.ipynb` (기초: 멀티모달 벡터 검색)

이 랩은 **"데이터 준비 $\rightarrow$ 임베딩(벡터화) $\rightarrow$ 인덱싱 $\rightarrow$ 검색 $\rightarrow$ 최적화"**의 흐름으로 진행됩니다.

#### **[코드 블록 분석]**
* **1단계: 환경 설정 및 연결**
    * 필요한 라이브러리(`pymongo`, `voyageai` 등)를 임포트합니다.
    * `MONGODB_URI`를 사용하여 MongoDB Atlas 클러스터에 연결하고 상태를 확인(`ping`)합니다.
    * `Voyage AI` API 키를 설정하여 임베딩 모델을 사용할 준비를 합니다.

* **2단계: 데이터 수집 및 적재**
    * `books.json` 파일(책 정보: 제목, 줄거리, 표지 URL 등)을 로드합니다.
    * 기존 컬렉션(`books`)을 비우고, 로드한 데이터를 MongoDB에 대량 삽입합니다.

* **3단계: 멀티모달 임베딩 생성**
    * **이미지 임베딩:** 책 표지 이미지 URL을 가져와 `voyage-multimodal-3` 모델로 벡터화합니다.
    * **텍스트 임베딩:** 책 제목이나 줄거리 텍스트를 벡터화합니다.
    * **업데이트:** 생성된 임베딩(벡터값)을 MongoDB의 각 책 문서(Document)에 `$set` 연산자를 이용해 업데이트합니다.

* **4~5단계: 벡터 인덱스 생성**
    * MongoDB Atlas에 `vectorSearch` 인덱스를 생성합니다.
    * 필드 매핑: `embedding` 필드, 1024차원, `cosine` 유사도를 설정합니다.

* **6단계: 기본 벡터 검색 수행**
    * `vector_search` 함수 정의: 사용자 쿼리(텍스트/이미지)를 벡터화한 후 `$vectorSearch` 파이프라인을 실행합니다.
    * **시나리오 실행:** "금색 왕관을 쓴 남자" 같은 텍스트 쿼리로 책 표지 이미지를 검색하거나, 다른 이미지 URL로 유사한 표지를 검색합니다.

* **7단계: 필터링 (Pre-filtering)**
    * **메타데이터 활용:** 연도(`year`), 페이지 수(`pages`) 등의 필드를 인덱스 정의에 추가합니다.
    * **시나리오 실행:** "2002년 이후 출판된 책($gte)", "250페이지 이하($lte)" 등의 조건을 벡터 검색과 결합하여 정확도를 높입니다.

* **8단계: 최적화 및 하이브리드 검색**
    * **양자화(Quantization):** 벡터 크기를 줄여(`scalar` quantization) 메모리 효율과 검색 속도를 높이도록 인덱스를 수정합니다.
    * **하이브리드 검색 (Hybrid Search):** 키워드 검색(Full-text Search)과 벡터 검색을 결합하고, RRF(Reciprocal Rank Fusion) 알고리즘으로 결과 순위를 재조정하여 검색 품질을 극대화합니다.

---

### 2. `ai-rag-lab.ipynb` (응용: RAG 파이프라인 구축)

이 랩은 **"문서 청킹 $\rightarrow$ 컨텍스트 임베딩 $\rightarrow$ 지식 검색 $\rightarrow$ LLM 답변 생성 $\rightarrow$ 대화 기억"**의 흐름입니다.

#### **[코드 블록 분석]**
* **1~2단계: 환경 설정 및 데이터 로드**
    * MongoDB 연결 및 Voyage AI 설정.
    * `mongodb_docs.json` (MongoDB 기술 문서 데이터)를 로드합니다.

* **3단계: 청킹(Chunking) 및 임베딩**
    * **청킹:** `LangChain`의 `RecursiveCharacterTextSplitter`를 사용해 긴 기술 문서를 의미 단위(약 200 토큰)로 쪼갭니다.
    * **컨텍스트 임베딩:** `voyage-context-3` 모델을 사용하여 쪼개진 청크에 '입력 타입(document)' 정보를 포함한 벡터를 생성합니다.

* **4~5단계: 데이터 적재 및 인덱싱**
    * 임베딩된 청크 데이터를 `knowledge_base` 컬렉션에 저장합니다.
    * 벡터 검색을 위한 인덱스를 생성합니다.

* **6단계: 검색 및 필터링 고도화**
    * 기본 `vector_search` 함수를 구현하여 기술 관련 질문("백업 베스트 프랙티스")을 던져봅니다.
    * **메타데이터 필터링:** `productName`(제품명), `contentType`(문서 종류), `updated`(날짜) 등을 이용해 검색 범위를 좁히는 `$vectorSearch` 파이프라인을 작성합니다.

* **7단계: RAG (검색 증강 생성) 구현**
    * **프롬프트 엔지니어링:** 검색된 문서(Context)와 사용자 질문(Question)을 합쳐 LLM에게 전달할 프롬프트를 구성하는 함수(`create_prompt`)를 만듭니다.
    * **답변 생성:** LLM에게 프롬프트를 보내고 답변을 받아 출력합니다.

* **Re-ranking (재순위화)**
    * 검색된 결과(Top-K)를 `Voyage AI Reranker`에 다시 보내, 질문과의 관련성이 높은 순서대로 다시 정렬하여 LLM에게 더 정확한 문맥을 제공합니다.

* **8단계: 대화 메모리 (Chat History)**
    * **DB 저장:** MongoDB에 `session_id`, `role`(user/assistant), `content`, `timestamp`를 저장하는 구조를 만듭니다.
    * **문맥 유지:** 사용자가 질문할 때마다 과거 대화 내역(`retrieve_session_history`)을 불러와 LLM 프롬프트에 포함시킴으로써, "방금 내가 뭐라 그랬지?" 같은 후속 질문에 답변할 수 있게 만듭니다.

---

### 3. `ai-agents-lab.ipynb` (심화: LangGraph 기반 에이전트)

이 랩은 **"도구 정의 $\rightarrow$ 에이전트 판단 로직(Graph) $\rightarrow$ 자율 실행 $\rightarrow$ 상태 저장"**의 흐름입니다.

#### **[코드 블록 분석]**
* **1~3단계: 설정 및 이중 데이터 로드**
    * `full_collection`(문서 원본 - 요약용)과 `vs_collection`(임베딩 포함 - 검색용) 두 가지 컬렉션을 준비합니다.
    * 벡터 인덱스를 생성합니다.

* **4단계: 도구(Tools) 정의**
    * `@tool` 데코레이터를 사용해 에이전트가 사용할 함수를 정의합니다.
    * **Tool 1 (Q&A):** `get_information_for_question_answering` - 벡터 검색을 통해 특정 질문에 대한 답을 찾습니다.
    * **Tool 2 (요약):** `get_page_content_for_summarization` - 문서 제목으로 원본을 찾아와 내용을 반환합니다.

* **5~6단계: LLM 설정 및 도구 바인딩**
    * `GraphState`: 대화의 흐름(메시지 목록)을 저장할 상태 타입을 정의합니다.
    * **CoT 프롬프트:** LLM에게 "생각(Think)하고 도구를 사용하라"는 지시가 담긴 시스템 프롬프트를 주입합니다.
    * **Bind Tools:** LLM이 위에서 정의한 두 가지 도구의 존재를 알고, 필요할 때 호출할 수 있도록 연결합니다.

* **7~9단계: LangGraph 구성 (Nodes & Edges)**
    * **Agent Node:** 현재 상태를 보고 LLM을 호출하여, 답변을 할지 도구를 쓸지 결정합니다.
    * **Tool Node:** 에이전트가 도구 사용을 요청하면 실제로 해당 함수를 실행하고 결과를 반환합니다.
    * **Conditional Edge:** 에이전트의 응답에 `tool_calls`가 포함되어 있으면 `Tool Node`로, 아니면 종료(END)로 분기하는 로직(`route_tools`)을 구현합니다.
    * **Compile:** 위 노드와 엣지들을 연결하여 실행 가능한 그래프 앱(`app`)으로 컴파일합니다.

* **10단계: 에이전트 실행**
    * **시나리오 A:** "백업 방법 알려줘" $\rightarrow$ 에이전트가 Q&A 도구 선택 $\rightarrow$ 검색 결과로 답변.
    * **시나리오 B:** "이 문서 요약해줘" $\rightarrow$ 에이전트가 요약 도구 선택 $\rightarrow$ 문서 원본 가져와 요약.

* **11단계: 지속성 (Persistence)**
    * **Checkpointer:** `MongoDBSaver`를 사용하여 에이전트의 대화 상태(State)를 MongoDB에 저장합니다.
    * `thread_id`를 통해 사용자의 세션을 유지하며, 에이전트가 이전 대화의 맥락을 기억하고 작업을 이어나가는 것을 확인합니다.