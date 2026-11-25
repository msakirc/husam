# Hüsam — Single-call LLM worker

> **Part of the KutAI app tier.** Split from the [KutAI](https://github.com/msakirc/kutai) monorepo (`packages/husam/`) for reading and reuse; it calls into the KutAI app (`src.*`) and runs only inside it. Decoupling is tracked in the monorepo.

> Full name *Kumandan Hüsamettin* ("Commander Hüsamettin"): one order, carried
> out, no deliberation.

[English](#english) · [Türkçe](#türkçe)

<a id="english"></a>

## Purpose

Most LLM work in KutAI is not agentic: a grader verdict, a summary, a classifier
label, an image generation. These are single calls. General Beckman admits them as
`raw_dispatch` tasks and the orchestrator hands each one to `husam.run`, which picks
a model (or honours the pick Beckman already made at admission), runs exactly one
call through the dispatcher primitive, and maps the result to the response dict
every caller expects.

**It does NOT** run an agent: no profile, no ReAct loop, no tool calls, no action
parsing, and no imports from `coulson`. It does **not retry** — transport retries live
in `coulson.react` (main work) and in Beckman's availability-retry path (overhead);
Hüsam surfaces the failure. It does not admit tasks or enforce quotas either; that
already happened in Beckman before the task reached it.

## Install

```bash
pip install "husam @ git+https://github.com/msakirc/husam.git"
```

It installs, but it only does useful work inside a KutAI checkout. For that, clone the [monorepo](https://github.com/msakirc/kutai) instead.

## Public API

```python
import husam

# The spec the orchestrator pump hands over.
resp = await husam.run({
    "kind": "overhead",                  # or "main_work"
    "mission_id": 12,                    # optional; enables the mission cost ceiling
    "preselected_pick": pick,            # optional in-memory fatih_hoca Pick from admission
    "context": {"llm_call": {
        "messages": [{"role": "user", "content": "..."}],
        "agent_type": "grader", "difficulty": 4,
        "call_category": "overhead",     # optional; falls back to "kind"
        "needs_json_mode": True,         # optional selection hints, all passed to fatih_hoca
        "estimated_output_tokens": 300,
    }},
})
# -> {"content", "model", "model_name", "cost", "usage", "tool_calls", "latency",
#     "thinking", "is_local", "ran_on", "provider", "task", ...}

# Image lane: same entry point, context carries image_call instead of llm_call.
await husam.run({"context": {"image_call": {
    "raw_dispatch": True, "prompt": "...", "width": 1024, "height": 1024,
    "quality_tier": "fast", "out_dir": "out/",
}}})
```

Failures raise `ModelCallFailed` (main work, including an empty model pool) or
`RuntimeError` (overhead).

## Architecture

```
Beckman admits raw_dispatch task (may attach a Pick)
        │
orchestrator pump ──► husam.run(task)
        │
        ├─ image_call? ──► fatih_hoca.select(needs_image) ──► paintress / clair_obscur
        │
        └─ llm_call   ──► preselected Pick, else fatih_hoca.select(...)
                          └─► LLMDispatcher.execute(pick, messages)   (one attempt)
                              └─► hallederiz_kadir CallResult ──► mapping.result_to_dict
```

## Dependencies

- `fatih_hoca` (selection), `hallederiz_kadir` (`CallResult`), `paintress` +
  `clair_obscur` (image lane), `dabidabi` (mission cost ceiling), `kara_kutu`
  (admission forensics), `nerd_herd`.
- App-tier imports, all lazy inside `run`: `src.core.llm_dispatcher` (the dispatcher
  primitive), `src.core.router.ModelCallFailed`, `src.core.in_flight`,
  `src.core.heartbeat`. Hüsam therefore only runs inside the KutAI app.
- `general_beckman` is a declared back-edge (lifecycle events), tracked in
  `tests/test_package_layering.py`.

## Gotchas

- **Honour the preselected pick.** When Beckman attached a `Pick`, admission has
  already gated pool pressure and reserved the slot; re-selecting would repeat those
  gates and can pick a different model than the one that was reserved.
- The image branch is checked **first**: an image task has no `llm_call`, and falling
  through to the text path would dispatch a text model.
- `context` may arrive as a JSON string (DB round-trip); `run` decodes it.

## Tests

```powershell
python -m pytest packages/husam/tests
```

## In KutAI

The orchestrator sends single-call tasks (grader, summarizer, classifier, image generation) straight to `husam.run()` instead of an agent loop. Source: [`packages/husam/`](https://github.com/msakirc/kutai/tree/main/packages/husam) in the [KutAI monorepo](https://github.com/msakirc/kutai), where this package is developed.

## Status and limitations

- App tier: it imports the KutAI app (`src.*`) and runs only inside it. It is published so the design can be read and reused.
- Version 0.1.0, part of a personal research project. Developed and tested on Windows 11 with Python 3.10; other platforms are untested.

## License

MIT. See [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE).

---
<a id="türkçe"></a>
## Türkçe

### Amaç

KutAI'deki LLM işlerinin çoğu ajan işi değildir: bir puanlayıcı kararı, bir özet, bir
sınıflandırma etiketi, bir görüntü üretimi. Bunlar tek çağrıdır. General Beckman
bunları `raw_dispatch` görevi olarak kabul eder; orkestratör her birini `husam.run`'a
verir. Hüsam bir model seçer (ya da Beckman'ın kabulde yaptığı seçime uyar), dispatcher
ilkeliyle tam bir çağrı yapar ve sonucu her çağıranın beklediği yanıt sözlüğüne
dönüştürür.

Hüsam **ajan çalıştırmaz**: profil, ReAct döngüsü, araç çağrısı, eylem ayrıştırma ve
`coulson` içe aktarımı yoktur. **Yeniden denemez** — taşıma yeniden denemeleri
`coulson.react` (ana iş) ve Beckman'ın erişilebilirlik yolunda (yardımcı iş) yaşar;
Hüsam hatayı yukarı iletir. Görev kabul etmez, kota uygulamaz; bunlar görev ona
ulaşmadan Beckman'da olmuştur.

### Kurulum

```bash
pip install "husam @ git+https://github.com/msakirc/husam.git"
```

Kurulur, ama yalnızca bir KutAI çalışma kopyasının içinde işe yarar. Bunun için [monorepo](https://github.com/msakirc/kutai)'yu klonlayın.

### Genel API

```python
import husam

resp = await husam.run({
    "kind": "overhead",                  # ya da "main_work"
    "mission_id": 12,                    # isteğe bağlı; görev maliyet tavanını etkinleştirir
    "preselected_pick": pick,            # isteğe bağlı; kabulden gelen fatih_hoca Pick
    "context": {"llm_call": {
        "messages": [{"role": "user", "content": "..."}],
        "agent_type": "grader", "difficulty": 4,
    }},
})
# -> {"content", "model", "cost", "usage", "latency", "is_local", "provider", ...}
```

Hatalar `ModelCallFailed` (ana iş, boş model havuzu dahil) ya da `RuntimeError`
(yardımcı iş) fırlatır.

### Dikkat Edilecekler

- **Önceden seçilmiş modele uy.** Beckman bir `Pick` eklediyse kabul havuz baskısını
  zaten denetlemiş ve yeri ayırmıştır; yeniden seçmek bu kapıları tekrarlar.
- Görüntü dalı **önce** denetlenir: görüntü görevinde `llm_call` yoktur.
- Hüsam uygulama katmanından (`src.core.*`) tembel içe aktarım yapar; yalnızca KutAI
  uygulaması içinde çalışır.

### Testler

```powershell
python -m pytest packages/husam/tests
```

### KutAI içindeki yeri

Orchestrator tek çağrılık görevleri (puanlayıcı, özetleyici, sınıflandırıcı, görüntü üretimi) ajan döngüsü yerine doğrudan `husam.run()`'a gönderir. Kaynak: [KutAI monorepo](https://github.com/msakirc/kutai) içinde [`packages/husam/`](https://github.com/msakirc/kutai/tree/main/packages/husam); paket orada geliştirilir.

### Durum ve sınırlamalar

- Uygulama katmanı: KutAI uygulamasını (`src.*`) import eder ve yalnızca onun içinde çalışır. Tasarımı okunabilsin ve yeniden kullanılabilsin diye yayınlandı.
- Sürüm 0.1.0, kişisel bir araştırma projesinin parçası. Windows 11 ve Python 3.10 üzerinde geliştirilip test edildi; diğer platformlar denenmedi.

### Lisans

MIT. [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE) dosyasına bakın.
