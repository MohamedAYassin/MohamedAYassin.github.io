---
title: "Migrating Databases with Zero Downtime"
date: 2026-03-31
draft: false
author: "Mohamed"
tags:
  - Databases
  - Migration
  - DevOps
  - Kubernetes
  - System Design
image: /images/zero_downtime_db_migration.png
description: "مش معني أنه هيتم migration من داتابيز لداتابيز أنه يحصل downtime أو داتا تضيع في الprocess. دليل عملي لكيفية نقل قواعد البيانات بدون أي توقف."
toc: false
rtl: true
---

مش معني أنه هيتم migration من داتابيز لداتابيز أنه يحصل downtime أو داتا تضيع في الprocess.

في الblue green deployment الموضوع بيبقا سهل جدًا خصوصًا لو بتستعمل حاجة زي Kubernetes. خلينا نسمي الversion اللي قبل الmigration بblue واللي بعد الmigration بgreen.

بس قبل ما ننقل أي traffic للgreen، فيه مشكلة أساسية لازم نحلها:

الblue شغال والـ users لسه بيعملوا requests، وبالتالي الdatabase القديمة لسه بتجيلها reads و writes طول الوقت.

فلو عملنا migration مرة واحدة من الblue DB للgreen DB، وبعدها حولنا الـ traffic، عندنا window صغيرة جدًا ممكن فيها data تتغير في الـ blue ومتوصلش للgreen.

وهنا ممكن يحصل inconsistency أو حتى data loss.

عشان كده أول خطوة مش إننا ننقل الـ application نفسه، أول خطوة إننا نجهز الـ green database ونخليها continuously in sync مع الـ blue database.

يعني وإنت لسه شغال على الblue، تبدأ تنقل الexisting data للgreen، وفي نفس الوقت أي changes جديدة بتحصل على الblue لازم تفضل بتوصل للgreen.

وده ممكن يتعمل بأكتر من طريقة حسب نوع الdatabase والarchitecture اللي عندك.

ممكن تستخدم native database replication، ممكن تستخدم CDC بحيث كل change بتحصل على الsource تتبعت للtarget، وممكن في بعض الarchitectures تعمل dual-write بحيث الapplication يكتب في الاتنين.

الفكرة مش إيه الimplementation اللي استخدمته، المهم إن الgreen database متبقاش مجرد snapshot قديمة من الـ blue.

لازم تكون caught up مع كل الchanges اللي بتحصل.

والحته دي أهم مما شكلها يبان، لأن مجرد إنك تعمل initial data copy وتلاقي إن عدد الrows متساوي مش معناها إن الداتابيزين in sync.

ممكن تكون نقلت 20 مليون record، لكن أثناء النقل حصل 5000 update و2000 insert و1000 delete على الـ blue.

فلازم يبقا عندك mechanism بيلاحق الchanges دي ويطبقها على الـ green.

وبعدها تبدأ تراقب الreplication lag لحد ما توصل لمرحلة إن الـ green database almost أو completely caught up.

وفي نفس الوقت لازم الـ schema نفسها تكون backward compatible.

لأن أثناء الـ migration ممكن يبقا عندك blue وgreen شغالين في نفس الوقت، فمينفعش تعمل breaking schema change وتفترض إن الold application خلاص مش موجود.

يعني مثلًا لو الgreen محتاج column جديد، في الغالب تضيف الcolumn الأول بشكل backward-compatible، وبعدها تخلي الgreen يستخدمه، وبعد ما تتأكد إن الblue خلاص مش محتاجه تقدر تشيله في migration منفصلة.

بعد ما الdatabase الجديدة تبقا synced والschema متوافقة، هنا بقى نبدأ الـ actual rollout.

تعمل provisioning للgreen instances، وتبدأ تشغل الgreen application جنب الـ blue.

في البداية الblue هو اللي عليه تقريبًا كل الtraffic، والgreen ممكن ياخد نسبة صغيرة جدًا من الtraffic.

مثلاً 1%، وبعدها 5%، 10%، وهكذا.

وخلال المرحلة دي تبدأ تراقب الـ error rate، latency، database errors، application logs، والbehavior بتاع الgreen.

ولو عندك read traffic ممكن الموضوع يبقا أسهل لأن بعض الreads تقدر توجهها للgreen من غير ما تغير الwrites بالكامل.

لكن الwrites هي الجزء الحساس.

لازم يبقا واضح مين هو الsource of truth أثناء الtransition، وإزاي الwrites هتفضل consistent.

لأن أسوأ حاجة تعملها إن الblue يكتب في DB والgreen يكتب في DB تانية من غير ما يبقا عندك strategy واضحة لمزامنة الاتنين.

هنا بقى بييجي دور الcutover.

لما تتأكد إن الgreen caught up، والreplication lag تقريبًا صفر، والgreen application شغال كويس، تقدر تبدأ تحول الـ traffic.

وده ممكن يتم عن طريق load balancer، reverse proxy، service mesh أو أي layer عندك تقدر تعمل فيها traffic switching.

ممكن تعمل gradual traffic shift أو حتى تعمل switch سريع جدًا للـ green، حسب ال architecture والrisk اللي مستعد تتحمله.

بعد الcutover، الgreen هو اللي بقا serving production traffic، والblue بيتساب لفترة ك rollback path بدل ما تمسحه على طول.

وده مهم جدًا لأن لو اكتشفت بعد الmigration إن فيه bug في الgreen، تقدر ترجع الtraffic للblue بدل ما تكون خلاص دمرت الold environment.

بس الrollback نفسه محتاج يتخططله من قبلها.

لأن لو الgreen بدأ يعمل writes بعد الـ cutover، ورجعت فجأة للblue، مينفعش تعتمد إن الblue عنده نفس الداتا إلا لو كنت محافظ على replication في الاتجاهين أو عندك strategy واضحة للتعامل مع الwrites اللي حصلت بعد الcutover.

يعني الmigration مش مجرد:

copy database → deploy new app → switch traffic.

هي أقرب لـ:

prepare the new schema → copy existing data → continuously replicate changes → verify consistency → deploy green → gradually shift traffic → complete cutover → keep blue available for rollback.

وبكده تقدر تعمل database migration من غير ما توقف الsystem، ومن غير ما تخلي الusers حتى يحسوا إن حصل migration أصلًا.

والblue-green هنا مش هو اللي بيعمل الـ database migration.

هو بيديك طريقة تشغل الold وnew versions جنب بعض وتتحكم في الtraffic بينهم، بينما الreplication / CDC / synchronization هو اللي بيضمن إن الداتا تفضل consistent أثناء الtransition.