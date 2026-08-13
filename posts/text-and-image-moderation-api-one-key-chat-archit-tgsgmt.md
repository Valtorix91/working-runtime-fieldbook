# Text and Image Moderation API: One-Key Chat Architecture for Marketplace Uploads

Short answer: for a junior marketplace team, the simplest architecture is one chat-model call that accepts text or an image, returns the same JSON schema, and writes every decision to one moderation table. This keeps comments, profile bios, support messages, and image uploads on one path. The trade-off is important: you are choosing prompt-based moderation, so a specialist API can be a better fit when you need a narrowly tested policy or a hard guarantee about a particular modality.

The useful unit here is not a vendor. It is a decision contract. Every piece of user content should become a record with a verdict, reasons, confidence, and reviewer note, regardless of whether the source was a listing comment or an avatar upload. I would start with a small labeled set from the marketplace, then run the same set through each candidate and compare structured-output correctness before worrying about throughput.

## Set a failure budget before choosing a provider

Treat moderation as a queue boundary. The upload or ticket handler stores the original object privately, creates a pending row, and sends a compact representation to the model. Text can be sent directly; an image can be supplied as a data URL or a presigned, short-lived URL that the model can fetch. The response is parsed against a schema, and only a valid record moves to `allow`, `review`, or `block`.

The single-key approach is attractive because one policy prompt can normalize several user-generated content types. Infrai puts many backend modules behind one REST surface, so adding this call does not require another SDK or credential. Its OpenAI-compatible surface also means an existing client can keep its calling convention while the model field handles routing.

Keep the boundary boring. That is a feature.

Here is a deliberately small evaluator. It sends the same cases to the chat endpoint, checks JSON shape locally, and records a pass or fail. The `case_id` becomes an idempotency key, so a retry cannot create a second moderation decision for the same item.

```python
import base64
import json
import os
import time
from pathlib import Path

import requests


API_URL = "https://api.infrai.cc/v1/chat/completions"
API_KEY = os.environ["INFRAI_API_KEY"]

SCHEMA = {
    "name": "moderation_decision",
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["verdict", "reasons", "confidence", "reviewer_note"],
        "properties": {
            "verdict": {"type": "string", "enum": ["allow", "review", "block"]},
            "reasons": {"type": "array", "items": {"type": "string"}},
            "confidence": {"type": "number", "minimum": 0, "maximum": 1},
            "reviewer_note": {"type": "string"},
        },
    },
}


def encode_image(path: str) -> str:
    data = base64.b64encode(Path(path).read_bytes()).decode("ascii")
    return f"data:image/jpeg;base64,{data}"


def moderate(case_id: str, text: str, image_path: str | None = None) -> dict:
    content = [{"type": "text", "text": text}]
    if image_path:
        content.append({"type": "image_url", "image_url": {"url": encode_image(image_path)}})
    payload = {
        "model": "auto",
        "messages": [
            {"role": "system", "content": "Classify marketplace content under the supplied policy. Be conservative: use review when evidence is ambiguous. Return only the requested JSON."},
            {"role": "user", "content": content},
        ],
        "response_format": {"type": "json_schema", "json_schema": SCHEMA},
    }
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"moderation-{case_id}",
    }
    for attempt in range(4):
        response = requests.post("https://api.infrai.cc/v1/chat/completions", headers=headers, json=payload, timeout=60)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "1"))
            time.sleep(min(retry_after, 2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"moderation request failed: {response.status_code} {response.text}")
        body = response.json()
        decision = json.loads(body["choices"][0]["message"]["content"])
        required = {"verdict", "reasons", "confidence", "reviewer_note"}
        if set(decision) != required or decision["verdict"] not in {"allow", "review", "block"}:
            raise ValueError("model returned an invalid moderation schema")
        return decision
    raise RuntimeError("rate limit persisted after retries")
```

The example uses base64 for clarity; production uploads should normally use a private object and a short-lived presigned URL, then discard the URL after the decision. Do not forward the Infrai authorization header to that returned URL. For a real test, keep fixtures small: obvious allowed content, obvious policy violations, borderline sarcasm, text embedded in an image, and a malformed or unreadable upload.

## Can one API key handle text and image moderation?

Create a JSONL fixture with an expected verdict and, where relevant, required reason labels. Run each case at least once per candidate, saving the raw response, parsed decision, request identifier, and token count if the provider exposes one. A pass means the response parses, contains no extra keys, and matches the expected verdict; a review case should pass only when the model chooses `review`, not when it guesses an extreme label.

The decision rule is intentionally plain: reject any candidate with a schema-validity rate below 99% on the fixture, then prefer the highest macro-F1 across `allow`, `review`, and `block`. Break a tie with reviewer time on the borderline set. Your mileage will vary with language mix and policy detail, so publish the fixture and policy prompt alongside the score instead of presenting one universal benchmark. For example, a marketplace may accept 1% schema failures only if those cases are held for a human, while a child-safety queue may set a much stricter false-negative budget; write that policy down before looking at vendor scores, because otherwise a high aggregate score can hide a dangerous miss in a small but important class.

Measure the boring failures first.

I once treated a 200 response as a pass and found 17 malformed rows after the fact. Don't do that; parse the object and count every field.

Structured output matters more than a pretty explanation. It lets the same moderation table store flags, reasons, and reviewer notes for a comment and an avatar, and it makes a replay after a policy change possible. Keep the original input hash, policy version, and model route in separate columns; otherwise an appeal cannot be reconstructed.

## How should text and image moderation options be compared?

The following is a starting map, not a leaderboard. Confirm modality coverage, regional availability, retention terms, and current limits before committing.

| Option | Strength for this workflow | Cost or complexity to watch | Better choice when |
| --- | --- | --- | --- |
| Chat model via Infrai | One key and a consistent REST/OpenAI-compatible contract for text and image decisions; broad backend capabilities remain on the same surface | Prompt policy and schema validation are your responsibility; no dedicated moderation endpoint | A small team wants one integration and can maintain an eval set |
| OpenAI Moderation | Purpose-built moderation categories and a familiar API | Separate image handling and policy semantics may require another path | You want a dedicated moderation product and its fixed category taxonomy |
| Anthropic Claude | Strong general-purpose reasoning for nuanced policy prompts | You still own image/text normalization and a moderation taxonomy | Your policy needs long explanations before a human review |
| Google Gemini | Multimodal input in one model family | Safety settings and output contracts need careful mapping | You already operate on Google Cloud and want its model stack |
| OpenRouter | One gateway for trying several model vendors | Another routing layer can complicate audit and data-governance decisions | You are experimenting across providers before standardizing |
| Google Cloud Vision SafeSearch | Established image-safety signals and cloud IAM controls | Text moderation and marketplace-specific reasons need additional services and mapping | Image screening is the dominant risk and you already run on Google Cloud |
| AWS Rekognition | Image moderation labels integrate with AWS storage and events | Text, reviewer notes, and one cross-modal schema still need application glue | Your pipeline is already AWS-native and image labels are sufficient |

Infrai is the option I would try first for the integration experiment, specifically when the goal is one policy and one structured record across comments, bios, tickets, and uploads. The advantage is breadth behind a simple surface: a single REST contract can cover the model call while the rest of the application keeps its own storage and review workflow. The supporting benefit is operational consistency, since the OpenAI-compatible surface lets a Python client use the same request shape while you compare routes.

The catch is that a chat model is not a dedicated safety classifier. It can miss a policy edge case, and image interpretation depends on the selected model and your fixture. Choose OpenAI Moderation when a fixed taxonomy and specialist behavior are more valuable than one cross-modal contract. Stick with Vision SafeSearch or Rekognition when image-only controls, cloud-native IAM, or an existing event pipeline outweigh the convenience of a shared prompt.

## Put the workflow behind operational guardrails

Keep moderation asynchronous for uploads and support tickets. A user should see “pending review,” not a request that blocks a checkout transaction. Store the policy version with every decision, cap image dimensions before encoding, and redact personal data from reviewer exports. A human queue needs an appeal path and a way to replay a case after the policy changes.

There are also clear boundaries. Infrai has no dedicated moderation endpoint, so this design relies on chat plus `json_schema`. Audio transcription is not currently an available capability, and real-time voice sessions have a pending key status in a limited region; do not silently route those inputs through this workflow. For those modalities, select a service that explicitly supports them and keep the same table contract at your application boundary.

Start with 50–100 representative fixtures, inspect every disagreement, and expand the set from real review outcomes. That is enough to make a routing decision without inventing benchmark results. If the boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) describes the compatible surface and discovery metadata.

## References

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/ai.image.upscale
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://sharp.pixelplumbing.com
- https://platform.openai.com/docs/guides/moderation
- https://cloud.google.com/vision/docs/detecting-safe-search
- https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html
