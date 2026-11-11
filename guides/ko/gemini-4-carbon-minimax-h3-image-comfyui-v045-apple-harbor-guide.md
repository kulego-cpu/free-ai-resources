---
title: "24GB GPU에서 H3-Image 돌리는 법과 Gemini 4 Carbon 선택 기준: 용량부터 줄이고 Harbor용 맥 지름은 보류하라"
description: "Hugging Face 파일 목록과 ComfyUI v0.45.0 PR 기록을 대조해 135.32GB MiniMax-H3-Image를 29.41GB 로컬 조합으로 줄이는 조건을 정리했습니다. Gemini 4 Carbon 프로모션 종료 후 과금 구조, 벤치마크 약점, Apple Claris Harbor 관망 이유까지 짚어봅니다."
pubDate: 2026-11-11
author: "샤오커 [小可]"
category: "엔지니어링 분석"
tags: ["Gemini 4 Carbon", "MiniMax-H3-Image", "ComfyUI", "Apple Harbor", "VRAM 최적화"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


어젯밤 Hugging Face에서 `MiniMaxAI/MiniMax-H3-Image` 저장소 파일 트리를 열어보고 용량을 합산하다가 창을 닫을 뻔했습니다.

33.1B DiT 본체가 BF16 기준 12개 샤드로 나뉘어 있을 뿐 아니라, 같은 저장소 안에 `transformer_diffusers/` 폴더와 VAE 2종이 통째로 중복 탑재되어 있습니다. 게다가 66.7 GB짜리 텍스트 인코더는 옆 저장소인 `MiniMaxAI/MiniMax-H3`에 따로 걸려 있습니다. 구조 확인 없이 `git clone`을 쳤다가는 첫 이미지 한 장 뽑기도 전에 SSD 200 GB 이상이 증발합니다.

같은 11월 10일에는 서로 맞물리는 굵직한 소식이 여럿 터졌습니다. Google DeepMind는 `Gemini 4 Carbon` 유료 개발자 API를 전면 개방했고, ComfyUI는 `v0.45.0`에 맞춰 H3-Image 경량화 지원 PR `#17655`를 병합했으며, 9to5Mac은 Apple 산하 Claris에서 테스트 중인 로컬 추론 클러스터 앱 `Harbor`를 포착했습니다.

결국 개발자가 따져야 할 계산서는 두 장입니다. 클라우드 모델의 숨은 추론 토큰 소모를 어떻게 막을지, 그리고 100 GB급 오픈 가중치를 데스크톱 GPU에 어떻게 밀어 넣을지입니다. 아래 내용은 모두 공식 블로그와 저장소 메타데이터, 병합된 PR 내역만 대조한 수치이며, 직접 베어메탈 장비에서 초 단위 벤치마크를 재지 않았으므로 가짜 생성 시간은 적지 않겠습니다.

## 135GB H3-Image 딜레마: ComfyUI v0.45.0 양자화 없이는 일반 GPU 구동 불가

MiniMax는 별도의 가벼운 이미지 아키텍처를 짜지 않았습니다. 기존 H3-Omni 백본에서 오디오 분기만 떼어내어 텍스트 기반 생성과 다중 이미지 편집을 단일 체크포인트로 합쳤습니다.

공식 `README.md`에 명시된 원본 가중치 크기는 상당히 무겁습니다:

- DiT 백본은 33.1B 파라미터, 50층 단일 스트림 구조에 히든 차원 5376 사양입니다. BF16 가중치만 66.28 GB이고 임베딩·출력 레이어는 FP32를 유지하며 12 MB짜리 `pdd_fused_heads.safetensors`가 동봉됩니다.
- 텍스트 인코더는 `Qwen3-VL-32B`를 그대로 가져다 50번째 레이어 은닉 상태를 읽습니다. 이 폴더 하나만 66.7 GB입니다.
- 이미지 VAE는 CNN 인코더와 24층 ViT 디코더 조합으로 16배 공간 압축을 수행하며 FP32 기준 2.34 GB를 차지합니다.

순수 추론 가중치만 합쳐도 **135.32 GB**입니다. 다운로드할 때는 중복되는 Diffusers 미러 디렉토리를 반드시 빼고 받아야 합니다:

```bash
hf download MiniMaxAI/MiniMax-H3-Image --exclude "transformer_diffusers/*" "vae_diffusers/*"
hf download MiniMaxAI/MiniMax-H3 --include "text_encoder/*"
```

체급은 거대하지만 샘플링 횟수는 극단적으로 짧습니다. 모델 카드 기준 **6스텝 추론**으로 증류되었고 CFG를 제거해 `guidance_scale`이 1로 고정됩니다. 업스케일러 없이 네이티브 1600만 화소급 해상도를 한 번에 출력하며 1:1 비율은 4096x4096, 21:9 비율은 6272x2688까지 생성합니다.

아래는 공식 저장소에 올라온 로컬 SGLang 서버와 호스팅 API의 동일 시드 출력 비교 이미지입니다:

<img src="/images/blog/minimax-h3-image-local-vs-api-2026.webp" alt="MiniMax H3-Image 로컬 SGLang 추론과 공식 API 4736x3552 출력 비교" />

실제 내부 파이프라인에 올릴 때는 두 가지 함정을 먼저 피해야 합니다.

첫째는 엄격한 파라미터 검증입니다. SGLang이나 vLLM-Omni 기반 `/v1/images/generations` 엔드포인트에서는 `num_inference_steps`를 6으로 주거나 생략해야 하고 `guidance_scale`도 1 외의 값을 받지 않습니다. 평소 습관대로 20스텝을 던지면 백엔드가 즉시 `HTTP 400`을 뱉습니다. `negative_prompt` 역시 완전히 차단되므로 피하고 싶은 요소는 긍정 프롬프트 문장에 풀어 적어야 합니다.

둘째는 라이선스 지역 제한 조항입니다. 루트 경로 `LICENSE` 1조 5항과 4조를 열어보면 **유럽연합·영국·한국·미국**이 제외 적용 지역인 Excluded Territories로 못 박혀 있습니다. 적용 가능 지역이라도 연 매출 100만 달러를 넘기거나 제3자 대상 호스팅 API를 열려면 `platform.minimax.io/console/license`를 통해 사전 서면 허가를 받아야 합니다. 상용 서비스 적용 전 법무 검토가 필수인 대목입니다.

그렇다면 24 GB~32 GB 일반 GPU에서는 어떻게 돌릴 수 있을까요? 출시 당일 배포된 **ComfyUI v0.45.0**이 돌파구를 열었습니다. 개발자 `@kijai`가 올린 PR `#17655`가 병합되면서 오디오 없는 `MiniMaxH3Image` 모델 클래스, PDD 스케줄러, 최대 16장의 `<Picture N>` 참조 이미지를 받는 `MiniMaxH3ImageConditioning` 노드가 추가되었습니다.

Comfy-Org 미러 저장소도 소비자용 GPU에 맞춰 가지치기 및 양자화 파일을 패키징해 올렸습니다:

| 모델 구성 요소 | 미러 저장소 파일명 | 양자화 형식 | 파일 크기 | 로컬 배치 경로 |
| --- | --- | --- | --- | --- |
| DiT 무손실 트리밍 | `minimax_h3_image_pruned_bf16.safetensors` | Pruned BF16 | 40.23 GB | `models/diffusion_models/` |
| DiT 균형형 | `minimax_h3_image_pruned_int8_convrot.safetensors` | Pruned INT8 | 20.98 GB | `models/diffusion_models/` |
| DiT 최소 용량 | `minimax_h3_image_pruned_w4a8.safetensors` | Pruned W4A8 | 12.55 GB | `models/diffusion_models/` |
| 텍스트 인코더 INT8 | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | INT8 | 27.14 GB | `models/text_encoders/` |
| 텍스트 인코더 FP4 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | NVFP4 AWQ | 15.69 GB | `models/text_encoders/` |
| 경량 프롬프트 확장기 | `qwen3.5_9b_w4a8_gptq.safetensors` | W4A8 GPTQ | 7.34 GB | `models/text_encoders/` |
| 이미지 VAE | `minimax_h3_image_vae_fp16.safetensors` | FP16 | 1.17 GB | `models/vae/` |

가장 가벼운 조합만 합치면 **29.41 GB**입니다. PR `#17655` 테스트 기록을 보면 의문이 하나 남습니다. `@kijai`는 단일 RTX 5090에서 BF16과 INT8 기준 1024 및 2048 해상도 생성을 검증했는데, 32 GB VRAM에 어떻게 40.23 GB짜리 BF16이나 도합 48 GB를 넘는 INT8 세트가 올라갔을까요?

비결은 단계별 언로딩과 CPU 오프로드입니다. 텍스트 인코더 연산이 끝나면 VRAM에서 바로 내리고 DiT를 불러와 6스텝 Euler `simple` 샘플링을 돌리며 초과 블록은 PCIe를 타고 시스템 RAM과 페이징을 주고받은 뒤 마지막으로 1.17 GB VAE로 디코딩합니다. 따라서 시스템 RAM이 64 GB 미만인 PC에서 BF16을 억지로 돌리면 페이징 병목으로 멈춰 섭니다. 24 GB 그래픽카드라면 무조건 W4A8 조합부터 잡는 편이 낫습니다.

다중 이미지 편집 시 해상도를 키우다 OOM이 뜬다면 캔버스 크기부터 낮춰야 합니다. 공식 텍스트 투 이미지 템플릿이 16 MP 대신 4 MP인 2048x2048을 기본값으로 깔아둔 데는 이유가 있습니다. `Resize Image/Mask` 노드를 켜서 참조 이미지 변 길이를 제한하면 잦은 VRAM 초과를 막을 수 있습니다. 또한 `v0.45.0`에는 Qwen 3.x 병렬 선형 가중치 병합 가속 `#17361`, FP8 텍스트 크래시 수정 `#17620`, SeedVR2 가용 VRAM 연동 VAE 슬라이싱 `#17508`도 들어갔으니 기존 환경도 판올림할 가치가 충분합니다.

## Gemini 4 Carbon 비용과 성능 손익표: 긴 문맥은 압도적이지만 일반 코딩 라우팅은 낭비

10월 28일 사이버 보안 방어 프로그램인 Fairwind로 먼저 풀렸던 `Gemini 4 Carbon`이 11월 10일 AI Studio 유료 티어, Android Studio, Gemini for business, Google Antigravity 전체에 정식 공개되었습니다.

초기 프로모션 단가만 보면 Argon과 같아 상당히 싸 보입니다. 입력 100만 토큰당 2달러, 출력 10달러, 캐시 입력은 0.1달러입니다.

하지만 전체 API 트래픽을 넘기기 전에 각주 문구부터 확인해야 합니다. 프로모션 기간이 끝나면 단가가 정확히 두 배인 입력 4달러, 출력 20달러로 뜁니다. 게다가 진짜 무서운 변수는 내부 추론 토큰 소모량입니다. DeepMind 공지에도 장기 검증 루프 때문에 Argon보다 훨씬 많은 토큰을 쓴다고 적혀 있어, 단순 CRUD 수정까지 Carbon에 붙이면 과금 청구서가 금세 불어납니다.

DeepMind가 함께 공개한 공식 벤치마크 표는 강점과 한계를 꽤 솔직하게 보여줍니다:

<img src="/images/blog/gemini-4-carbon-benchmarks-2026.webp" alt="Gemini 4 Carbon과 Gemini 4 Argon, GPT-6 Astra, Claude Opus 5.5 공식 벤치마크 비교표" />

선택 기준은 명확합니다. `AA Index v5.0` 59점으로 장문 추적과 보안 방어는 탁월하지만 두 갈래 실무 트랙에서는 약점을 보입니다:

- 수십만 단어 단위 감사 보고서 추적과 보안 점검은 Carbon이 최적입니다. `GraphWalks` 256k~1M 구간에서 96.2% F1을 기록해 경쟁 모델을 20점 이상 따돌렸고, `SEC-bench Pro` 88.2%, Gray Swan 간접 프롬프트 인젝션 공격 성공률 0.0%, `Terminal-Bench 4.0` 치팅률 0.0%를 달성했습니다. 같은 날 프리뷰로 열린 Antigravity `/swarm` 워크플로에서 총괄 설계와 감사 역할에 두고 실무 코딩은 가벼운 워커 모델로 분산할 때 가성비가 가장 좋습니다.
- 복잡한 멀티파일 저장소 패치 작업은 무조건 Carbon으로 넘기지 마십시오. `FrontierSWE 2` 점수는 61.2%로 65.5%를 기록한 `GPT-6 Astra`보다 4.3%p 뒤집니다.
- 데스크톱 GUI 연속 조작 역시 마찬가지입니다. `OSWorld-2.1` 오프라인 서브셋에서 81.5%에 그쳐 `Claude Opus 5.5`의 87.2% 대비 5.7%p 낮습니다. 종합 순위 대신 작업 성격별로 API 라우터를 분리하는 편이 맞습니다.

## Apple Claris Harbor 유출: Thunderbolt 5 메모리 풀 보고 지금 맥 살 타이밍은 아니다

같은 날 `r/LocalLLaMA`에서 화제가 된 소식은 9to5Mac이 유출가 `@aaronp613`의 발견을 인용해 전한 Apple 내부 테스트 앱 **Harbor**입니다.

방향성은 터미널 설정을 아예 없애는 데 있습니다. Apple 자회사 Claris 브랜드 아래 개발 중인 Harbor는 GUI 안에서 오픈 가중치 모델을 검색·다운로드해 실행하면 Mac 한 대 또는 여러 대를 사설 AI 추론 서버로 전환해 줍니다. **Thunderbolt 5** 케이블로 Mac 두 대 이상을 엮으면 자동으로 피어를 인식해 통합 메모리를 하나의 공유 풀로 묶는 구조입니다.

앞에서 본 135 GB 무게를 떠올리면 Apple의 노림수가 바로 보입니다. 지금까지 66 GB급 텍스트 인코더나 100B 이상 MoE 모델을 무양자화로 집에서 돌리려면 Mac Studio 여러 대를 놓고 MLX 분산 링과 SSH 터널링을 직접 엮어야 했습니다. Harbor는 이 진입 장벽을 선 꽂는 수준으로 내리려 합니다.

다만 이를 근거로 지금 하드웨어 구매 버튼을 누르는 것은 이릅니다. Harbor는 아직 Claris 비공개 프리뷰 상태여서 정식 출시일도, OpenAI 호환 API 지원 범위도, 이미지 확산 연산자 최적화 여부도 전혀 공개되지 않았습니다. 또한 통합 메모리 클러스터가 만들어주는 것은 128 GB·256 GB급 '용량'이지 NVIDIA 텐서 코어의 연산 처리량이 아닙니다. 1600만 화소 DiT 같은 연산 집약 작업에서는 M 시리즈 GPU가 FP4 가속을 쓰는 RTX 5090보다 훨씬 느릴 가능성이 높으니 실제 Mac Studio 다중 연결 처리량 로그가 공개된 뒤에 결정해도 늦지 않습니다.

---

본문에 인용한 모델 크기, 벤치마크 수치, 라이선스 조항은 아래 1차 출처에서 직접 대조할 수 있습니다:

| 검증 항목 | 1차 공식 출처 URL |
| --- | --- |
| Gemini 4 Carbon 정식 출시 및 가격 각주 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/ |
| Gemini 4 Carbon 벤치마크 수치 원문 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/ |
| MiniMax-H3-Image 모델 카드 및 아키텍처 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image |
| MiniMax 라이선스 제외 지역 조항 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image/blob/main/LICENSE |
| ComfyUI PR #17655 병합 기록 | https://github.com/Comfy-Org/ComfyUI/pull/17655 |
| ComfyUI 공식 배포 가이드 및 템플릿 기준 | https://docs.comfy.org/tutorials/image/minimax/minimax-h3-image |
| Comfy-Org 양자화 가중치 파일 목록 | https://huggingface.co/Comfy-Org/MiniMax-H3-Image |
| Apple Claris Harbor 유출 보도 | https://9to5mac.com/2026/11/10/apple-has-new-mac-app-coming-to-make-running-local-ai-models-easy/ |

현재 주력 카드가 24 GB VRAM이라면 ComfyUI v0.45.0으로 올린 뒤 `Resize Image/Mask` 노드를 켜고 W4A8 조합의 호스트 메모리 스왑량부터 체크해 보십시오. 클라우드 쪽은 고난도 보안 감사와 긴 문맥 작업만 Carbon에 태우고 일반 코드 편집은 기존 저가 모델에 남기는 라우팅이 가장 현실적입니다. 지금 로컬 대형 모델을 돌릴 때 가장 먼저 발목을 잡는 병목은 절대적인 VRAM 용량인가요, 아니면 CPU 오프로드 시 발생하는 대역폭 지연인가요?

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.

