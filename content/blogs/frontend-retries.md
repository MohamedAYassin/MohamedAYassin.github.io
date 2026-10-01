---
title: "ليه بننسى الـ Frontend Retries؟"
date: 2026-10-02T00:00:00+03:00
draft: false
author: "Mohamed Yassin"
tags:
  - Frontend
  - Resilience
  - Architecture
  - Reliability
  - Distributed Systems
  - UX
description: "ليه بنفكر في الـ resilience بتاعت السيرفرات والـ backend وننسى الـ frontend؟ نظرة على هندسة الـ retries، التعامل مع الـ transient failures، ومخاطر الـ retry storms."
rtl: true
dir: "rtl"
toc: true
---

من الحاجات اللي بستغرب منها إن ناس كتير بتعمل **backend retries** وبتفكر طول الوقت في الـ resilience بتاعت الـ services، لكن الـ frontend نفسه أول ما request تفشل بيوري اليوزر كمية لون أحمر كأن جهازه هينفجر، لمجرد إنه مش معمول حسابه يعمل retry!

مع إن كتير من الـ request failures اللي بتحصل للـ user في الحقيقة مش failures حقيقية في السيستم.

---

## مش كل فشل بيكون فشل حقيقي (Transient Failures)

في العالم الحقيقي، في أسباب كتير جدًا تخلي الـ request تفشل لحظيًا:
- **انقطاع لحظي في الشبكة:** الـ network connection تقطع لثانية واحدة وترجع.
- **Request Timeout:** الـ timeout يحصل بسبب latency مؤقت أو بطء لحظي.
- **تبديل الشبكة:** الـ mobile ينتقل فجأة من Wi-Fi لـ 4G/5G.
- **أخطاء Gateway مؤقتة:** السيرفر يرجع أخطاء عابرة زي `502 Bad Gateway` أو `503 Service Unavailable`.
- **Server Spikes:** السيرفر يكون overloaded لمدة أجزاء من الثانية ويرجع يخدم الطلبات بعدها عادي جدًا.

وفي كل الحالات دي، أول request fail مش معناه إطلاقًا إن الـ operation فشلت فعلًا.

مثلًا تخيل السيناريو ده:

```text
Frontend  ──( GET /profile )──>  Network Hiccup  ──>  Timeout  ──>  [ ERROR ! ]
```

بدل ما الـ UI يعتبر إن الدنيا انهارت ويعرض شاشة حمراء للمستخدم، ممكن يعمل retry تلقائي بعد فترة صغيرة. وبدل ما الـ user يشوف error من أول محاولة، الـ application يقدر يتعامل مع الـ transient failure في الخلفية بهدوء وبدون ما يحسس اليوزر بأي مشكلة.

---

## هندسة الـ Retry: مش مجرد ضرب الـ Endpoint وخلاص

والـ retry مش معناه إنك تضرب نفس الـ endpoint 20 مرة ورا بعض لحد ما تشتغل!

المفروض يكون فيه limit واضح لعدد الـ retries، وغالبًا الأفضل تستخدم **Exponential Backoff مع Jitter** بدل ما كل الـ clients تعيد المحاولة في نفس اللحظة وتخنق السيرفر:

```text
1st attempt ──> fail
  └── wait ~200ms (+ jitter)
2nd attempt ──> fail
  └── wait ~500ms (+ jitter)
3rd attempt ──> fail
  └── Show meaningful error to user
```

---

## إيه الـ Errors اللي تستاهل Retry وإيه لأ؟

طبعًا مش كل نوع error يستحق إنك تعمله retry:

- **أخطاء الـ Client Errors (4xx):** لو السيرفر رجع `400 Bad Request`، أو `401 Unauthorized`، أو `403 Forbidden`، أو `404 Not Found`؛ إعادة نفس الـ request بنفس المعطيات مش هتصلح أي حاجة، لأن المشكلة إما في الـ input أو في الصلاحيات.
- **الأخطاء المؤقتة (Transient & 5xx):** الـ retry بيكون مفيد ومثالي جدًا مع الـ network errors، والـ timeouts، وبعض ردود الـ 5xx (زي `502` و `503` و `504`).

---

## الـ Resilience كـ Layers: مش بس في الـ Frontend

الموضوع كمان مش المفروض يقف عند الـ frontend؛ الـ resilience لازم تكون **Layered**:

- الـ **Frontend** ممكن يكون عنده retry strategy خاصة بتجربة المستخدم.
- الـ **Backend** أو الـ **Reverse Proxy** ممكن يكون عنده retry/failover strategy تانية للتعامل مع الـ microservices الداخلية.

ولو أنت شغال Kubernetes، فعندك أصلًا أكتر من replica للـ backend. لو instance معينة وقعت أو بقت `NotReady`، الـ Kubernetes يقدر يشيلها تلقائيًا من الـ endpoints ويوجه الـ traffic للـ healthy instances.

> **نقطة مهمة هنا:**
> وجود Kubernetes Service مش معناه أبدًا إنه لو request فشل وهو رايح لـ Backend 1، إن نفس الـ request هيتعاد أوتوماتيك لـ Backend 2.

الـ connection نفسها ممكن تفشل والـ user يستلم error في إيده. عشان كده في بعض الـ architectures بنضيف retry/failover layer في الـ backend أو في الـ **Service Mesh** (زي Istio أو Envoy) أو الـ Ingress Controller، بحيث لو حصل transient failure مع instance معينة، يقدر يحاول مع instance تانية بسرعة بدل ما يرجع error للمستخدم على طول.

---

## كابوس الـ Retry Storms

ممكن جدًا يكون عندك أكتر من layer للتعامل مع نفس الـ failure، لكن كل layer لازم تكون محسوبة بدقة بالغة.

ليه؟ لأنك لو عملت 3 retries في الـ frontend، و3 retries في الـ service mesh، و3 retries في الـ backend:

```text
1 User Request
  └── 3 Frontend Retries
        └── × 3 Service Mesh Retries
              └── × 3 Backend Retries
                    = 27 Total Requests!
```

طلب واحد بس من الـ user ممكن يتحول لعدد ضخم جدًا من الـ requests المتتالية، وده اللي بنسميه **Retry Storm**. ساعتها أنت بتزود الـ load بشكل كارثي على system أصلًا واقع أو بيعاني وبيحاول يتعافى!

---

## النقطة الأخطر: إياك تعمل Blind Retry لـ POST Requests

مينفعش إطلاقًا تعمل blind retry لأي طلب غير آمن زي `POST` أو طلبات تغيير الحالة (Non-idempotent operations).

تخيل معايا السيناريو ده:

```http
POST /payments
```

الـ payment اتنفذت في الـ database واتسحبت الفلوس من حساب العميل بالفعل، لكن الـ response ضاع في الـ network قبل ما يوصل للـ frontend.

هنا الـ frontend هيفهم إن العملية فشلت ويعمل retry تلقائي، وساعتها بدل عملية دفع واحدة ممكن تتحول لعمليتين، والعميل يتخصم منه الفلوس مرتين!

عشان كده الـ retry strategy لازم تكون متصممة ومبنية بالتنسيق التام مع الـ backend، باستخدام أدوات أساسية زي **Idempotency Keys** لما تكون العملية حساسة وممكن تتكرر.

---

## تجربة المستخدم (UX): بلاش تفجع اليوزر

الـ UX نفسها نقطة جوهرية ومهمة جدًا.

المستخدم مش مفروض أول ما تقع منه الشبكة لثانية واحدة يلاقي قدامه شاشة حمراء مكتوب فيها:

```text
REQUEST FAILED
TRY AGAIN
```

ببساطة شديدة:
- الـ UI ممكن يفضل في **Loading State** هادئة شوية.
- يعمل retry بهدوء في الخلفية.
- لو العملية فضلت تفشل بعد استنفاد كل المحاولات، ساعتها بس تعرض للمستخدم رسالة error واضحة ومفهومة، وتديله action محدد يقدر يعمله (زي زرار "إعادة المحاولة" بعد التأكد من اتصاله).

---

## الخلاصة: الـ Network غير موثوقة بطبيعتها

الفكرة الأساسية إن الـ network بطبيعتها **Unreliable**:
- الـ Nodes هتقع.
- الـ Pods هتموت.
- الـ Connections هيحصل لها Timeout.
- الـ Requests هتفشل.

ده مش exceptional behavior في الـ distributed systems، ده جزء طبيعي من الـ **normal operating conditions**.

فلو الـ frontend مبني على افتراض ساذج إن:

```text
request failed  ==  operation failed
```

فأنت حرفيًا بتحول مشكلة transient صغيرة وغير مرئية في الـ infrastructure إلى مشكلة مزعجة في وش المستخدم.

الـ application المفروض يكون ناضج ومرن كفاية إنه يتحمل الـ failures الصغيرة دي في صمت، بدل ما يحيي الـ user بيها كل مرة!
