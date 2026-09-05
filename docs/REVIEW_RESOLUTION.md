# Review resolution contract

Repository: Saber5656/dubstudio; PR #4

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTNkGBs6QA88t

### Let preset TTS skip voice_ref

**Normative resolution**

supports_cloning=falseのpreset TTSではvoice_refの取得・待機とvoice-cloning consentを要求しない。voice_refとconsentはcloning providerを選択した場合だけ必須とし、非cloning経路は通常のpreset voiceで完結させる。

**Focused verification before resolving this thread:**

openai-tts等の非cloning presetをvoice_refなし・consent未記録でsynthesizeし、成功経路へ進むことを確認する。cloning providerでは従来どおりvoice_ref不足とconsent不足が拒否されることも確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA88w

### Make the advertised default install include default providers

**Normative resolution**

default provider matrixとinstall profileを一致させる。core installだけでdefault pipelineが動くよう依存を含めるか、既定値をcoreで利用可能なproviderへ変更し、extrasを選ぶ場合はinitが明示的にprofileを要求してmissing-extraを早期に表示する。

**Focused verification before resolving this thread:**

clean virtualenvで広告どおりのdefault installからinit→run/planを実行し、provider missing errorが出ないことを確認する。各extra profileの依存解決とmatrixのprovider availabilityもCIで検査する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA88z

### Avoid accepting no-op project config fields

**Normative resolution**

project.input、source_language、target_languagesの権威をmanifestに一本化する。runtime configから同名のno-op overrideを削除するか、init時にmanifestへ同期して変更時に明示的な再生成/invalidationを要求し、pipelineが黙って旧manifestを使わないようにする。

**Focused verification before resolving this thread:**

config/envで各値を変更した場合に、拒否またはmanifest更新・再計画が必ず発生することを確認し、verify_input、graph、transcribeが同じeffective manifestを参照することをintegration testで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA881

### Ship the policy with wheel installs

**Normative resolution**

consent promptが参照するPOLICY.mdとversion historyをwheel/uvx installで読めるinstalled resourceとして同梱する。実行場所のrepository docsに依存せず、policy versionと表示内容が配布物に固定される。

**Focused verification before resolving this thread:**

sdist/wheelをclean environmentへinstallし、installed packageからPOLICY.md本文とversionを取得してpromptに表示できることを確認する。source checkout外でもconsent gateがmissing policyで失敗しないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA883

### Use the valid uv export format

**Normative resolution**

supply-chain workflowのuv exportはrequirements.txt形式を使い、pip-auditへの入力を明示的に生成する。存在しないrequirements-txt指定を除去し、uvの現行CLI contractとlockfileを同一jobで検証する。

**Focused verification before resolving this thread:**

CI runnerでuv export --format requirements.txtが成功し、生成requirementsをpip-auditが読み込むことを確認する。export失敗時にauditを成功扱いにしないことも確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA884

### Expose audio-stream selection during init

**Normative resolution**

initに--audio-streamを追加し、選択値をproject manifest/configへ保存してingestが同じeffective valueを使う。範囲外・非整数はinit時に拒否し、未指定は既定streamを明示的に選ぶ。

**Focused verification before resolving this thread:**

2本以上のaudio stream fixtureで--audio-stream 1をinitし、manifestとingestがstream 1を使うこと、0/範囲外/不正値が明確なvalidation errorになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA887

### Wire the force-retranslate option

**Normative resolution**

run/planのargument mappingに--force-retranslateを追加する。edited/approved translationを上書きする場合は対象と破壊性を表示して明示確認を要求し、flagなしでは既存編集を保持する。

**Focused verification before resolving this thread:**

edited/approved rowをfixtureにしてflagなしがskip、flagありのconfirmation拒否が無変更、confirmation承認だけが再翻訳することを確認する。planにも同じ対象と破壊性が表示されることを検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA889

### Redact secrets learned after logging starts

**Normative resolution**

redaction filterを起動時envのsnapshotだけにせず、configで読み込む.env secrets、provider keys、ui tokenを取得した時点でdynamic registryへ登録する。log sinkへ渡す前にregistry、Bearer、URL query等を用いて全出力をredactする。

**Focused verification before resolving this thread:**

logging初期化後に.envとui tokenをloadし、exception/access log、JSONL、/?token=、Bearer形式の各出力に秘密値が現れないことを確認する。registry更新の競合時も未redacted記録がないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA88_

### Handle ASR segments without log probabilities

**Normative resolution**

avg_logprobがNoneのsegmentは例外や一律rejectにせず、明示したneutral/unknown confidenceとしてscoreから除外または保守的な別scoreを適用する。voice-reference選定はmissing confidenceだけで全候補を失わない。

**Focused verification before resolving this thread:**

avg_logprob有り・None混在・全てNoneのASR responseを入力し、cloning policyに従った候補選定と説明可能なscoreが得られること、None比較例外がないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA89A

### Do not require workflow_dispatch before merge

**Normative resolution**

CodeQLのpre-merge gateはpull_request等、feature branchで実際に発火するtriggerのresultに依存する。workflow_dispatchはdefault branch反映後の手動検証として扱い、PR merge前の必須証跡にしない。

**Focused verification before resolving this thread:**

feature branchのPRでCodeQL jobが起動し、required checkとしてterminal successになることを確認する。default branch反映後のmanual dispatchは別のpost-merge validationとして記録する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA89C

### Keep MT base URLs TLS-safe

**Normative resolution**

MT providerのbase_urlはhttps://を要求し、例外は明示したlocalhost/loopback test endpointだけに限定する。Authorization headerは非TLSのremote endpointへ決して送らない。

**Focused verification before resolving this thread:**

https remote、http remote、https localhost、http localhost、IPv4/IPv6 loopbackをfixtureで検査し、許可境界とAuthorization送信抑止を確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA89H

### Provide a real default translation model

**Normative resolution**

translation.openai.modelにsupported modelの具体的な既定値を設定し、init template・config schema・provider matrixで同じ値を使う。空値やコメントだけの候補はinvalidとしてinit/run前に検出する。

**Focused verification before resolving this thread:**

clean init直後にmodelが解決されrun/planがschema errorにならないこと、明示したunsupported/empty modelは実行前に拒否されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA89J

### Base short-audio fitting on the slot, not bleed room

**Normative resolution**

short-audio fittingのslow-down/padding判定はsegmentのslot_msを基準にする。following gap/bleed roomはslotを超えた音声のoverflow処理だけに使い、slot内の音声を不必要に遅くしない。

**Focused verification before resolving this thread:**

synth_ms=slot_msで大きなgapがあるcase、synth_ms>slot_ms、synth_ms<slot_msをfixtureにし、ratioとpadding/slow-downがslot基準で期待どおりになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkGBs6QA89L

### Protect source-audio cache writes

**Normative resolution**

GET /api/audio/sourceのcache生成・lazy deleteはread-only routeの前提を破らない専用cache lockとatomic temp-file/rename、transcript generation keyを使う。runと並行するrequestがpartial fileや世代混在を読まないようにする。

**Focused verification before resolving this thread:**

同一sourceへの並列GET、transcript更新中のGET、lazy deleteとreadの競合を実行し、完全な旧/新fileだけが返り、partial/foreign generationやlost updateがないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
