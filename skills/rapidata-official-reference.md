# Rapidata SDK — Full API Reference

## Job Definition Parameters

The new job-definition API exposes **classification**, **comparison**, **locate**, **draw**, **select words**, **free text**, and **ranking** publicly.

### Common parameters (classification & comparison)

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | str | Job identifier (not shown to labelers) |
| `instruction` | str | Task description shown to labelers |
| `datapoints` | list | Data to label (URLs or local paths) |
| `data_type` | `"media"` \| `"text"` | `"media"` (default, covers image/video/audio) or `"text"` |
| `responses_per_datapoint` | int | Responses per item (default 10) |
| `contexts` | list[str] \| None | Text context per datapoint (max 400 characters each; the backend rejects longer values — set `rapidata_config.upload.autoShortenContext = True` to auto-shorten, or use `client.context` to shorten manually) |
| `media_contexts` | list[list[str]] \| None | Reference images per datapoint; each entry is a list of image URLs/paths (one inner list per datapoint) |
| `confidence_threshold` | float \| None | Confidence-based early stopping threshold (0-1); cannot combine with `quorum_threshold` |
| `quorum_threshold` | int \| None | Quorum-based early stopping: stop when this many responses agree; cannot combine with `confidence_threshold` |
| `settings` | `Sequence[RapidataSetting] \| None` | Display/behavior settings |
| `private_metadata` | `list[dict[str, str]] \| None` | Hidden metadata per datapoint |

### Classification-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `answer_options` | list[str] | Categories to choose from |

### Comparison-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Pairs: `[["a1.jpg","b1.jpg"], ...]` |
| `a_b_names` | list[str] \| None | Custom labels for results, e.g. `["Model A","Model B"]` |

### Locate-specific

Locate has no job-specific parameters — only the core parameters apply. `data_type`, `answer_options`, `a_b_names`, `confidence_threshold`, and `quorum_threshold` are not available for locate jobs. `datapoints` is `list[str]` (one item per row).

```python
job_definition = client.job.create_locate_job_definition(
    name="Artifact Detection",
    instruction="Tap on any visual glitches or errors in the image.",
    datapoints=["image1.jpg", "image2.jpg"],
    responses_per_datapoint=35,
    contexts=["Optional context"],
    settings=[LocateMaxPointsSetting(5)],
)
```

For locate audience examples, use `audience.add_locate_example(instruction, datapoint, truths, context=None, media_context=None, explanation=None, settings=None)` where `truths` is a `list[Box]` (import `Box` from `rapidata`); coordinates are image ratios (0.0–1.0).

### Draw-specific

Draw has no job-specific parameters — only the core parameters apply. `data_type`, `answer_options`, `a_b_names`, `confidence_threshold`, and `quorum_threshold` are not available for draw jobs. `datapoints` is `list[str]`.

```python
job_definition = client.job.create_draw_job_definition(
    name="Object Marking",
    instruction="Color in all the blue books",
    datapoints=["image1.jpg", "image2.jpg"],
    responses_per_datapoint=35,
)
```

For draw audience examples, use `audience.add_draw_example(instruction, datapoint, truths, explanation=None, settings=None)` where `truths` is a `list[Box]` (import `Box` from `rapidata`); coordinates are image ratios (0.0–1.0).

### Select Words-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `sentences` | list[str] | One sentence per datapoint, split by spaces for labelers to select words from (must have same length as `datapoints`) |

`contexts`, `media_contexts`, `data_type`, `answer_options`, `a_b_names`, `confidence_threshold`, and `quorum_threshold` are not available for select words jobs.

```python
job_definition = client.job.create_select_words_job_definition(
    name="Prompt Alignment",
    instruction="Select the words that are not depicted in the image.",
    datapoints=["image1.jpg", "image2.jpg"],
    sentences=["A cat on a red couch [No_mistakes]", "A blue car in the rain [No_mistakes]"],
    responses_per_datapoint=15,
)
```

For select words audience examples, use `audience.add_select_words_example(instruction, datapoint, sentence, truths, explanation=None, settings=None)` where `truths` is a `list[int]` of 0-based word indices to select.

### Free Text-specific

Free Text has no job-specific parameters — only the core parameters apply. `answer_options`, `a_b_names`, `confidence_threshold`, and `quorum_threshold` are not available. Note: free text answers cannot be graded against a ground truth; audiences cannot be trained with free text qualification examples.

```python
job_definition = client.job.create_free_text_job_definition(
    name="Prompt Collection",
    instruction="What would you like to ask an AI?",
    datapoints=["image1.jpg"],
    responses_per_datapoint=15,
)
```

### Ranking (via `client.job.create_ranking_job_definition`, `client.order.create_ranking_order`, or `client.flow.create_ranking_flow`)

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Groups: `[["img1","img2","img3"], ...]`; each inner list is one independent ranking set |
| `comparison_budget_per_ranking` | int | Total comparisons per ranking group |
| `responses_per_comparison` | int | Responses per individual comparison (default 1); replaces `responses_per_datapoint` for ranking |
| `random_comparisons_ratio` | float | Ratio of random vs targeted comparisons (0-1, default 0.5) |

`responses_per_datapoint`, `answer_options`, `a_b_names`, `confidence_threshold`, and `quorum_threshold` are not available for ranking jobs.

## Audiences

**Goal.** An audience selects a specific group of annotators for a **specific task**. You tailor the pool to that task two ways — by training it on **qualification examples** (tasks with a known-correct answer; only labelers who answer them correctly are recruited) and/or by attaching **recruitment filters** (country, language, demographics). The point is to get the *right* annotators onto *that* task.

A task-specific audience is meant for that task and its repeated or scheduled runs — **not** for reuse on a different, unrelated task. The qualification examples encode what "good" means for the original task; once the task changes they no longer describe the work, so reusing the audience silently loses the quality it was built for. Create a new audience per distinct task. (The `global` audience is the exception: it's the generic baseline pool for tasks that need no special qualification.)

**Three kinds:**

| Kind | How to get it | When to use |
|------|---------------|-------------|
| global | `client.audience.get_audience_by_id("global")` | Instant, baseline quality, no setup |
| curated | `client.audience.get_audience_by_id("aud_MU1GZYoESyO")` (alignment) | Pre-trained on a domain |
| custom | `client.audience.create_audience(name=...)` + `add_*_example(...)` | You need labelers qualified on *your* task |

**Lifecycle / management:**

```python
# Create / fetch / discover
audience = client.audience.create_audience(name="Expert Evaluators", filters=None)
audience = client.audience.get_audience_by_id("global")            # or "aud_..." / any audience id
audiences = client.audience.find_audiences(name="", amount=10, page=1)   # your audiences, newest first

# Train a custom audience (min 3 examples before recruiting starts; every truth must be human-reviewed)
audience.add_classification_example(instruction=..., answer_options=[...], datapoint=..., truth=[...])
audience.add_compare_example(instruction=..., datapoint=[...], truth=...)
audience.add_locate_example(instruction=..., datapoint=..., truths=[Box(...)])     # requires: from rapidata import Box
audience.add_draw_example(instruction=..., datapoint=..., truths=[Box(...)])
audience.add_select_words_example(instruction=..., datapoint=..., sentence=..., truths=[1])
df = audience.get_examples(amount=10, page=1)                       # inspect examples (DataFrame)

# Manage
audience.update_name("New Name")
audience.update_filters([CountryFilter(["US"]), LanguageFilter(["en"])])  # audience-supported filters only
filtered = audience.filter([CountryFilter(["US"])])                 # slim subset, reuses the pool (no re-recruiting)
audience.delete()

# Use
job = audience.assign_job(job_def)                                  # start a job on the pool
jobs = audience.find_jobs(name="", amount=10, page=1)              # jobs assigned to this audience
```

Note: free-text answers can't be graded against a ground truth, so there is no `add_free_text_example` — custom audiences cannot be trained for free-text tasks.

## Demographic Filters

All filters are importable from the top-level `rapidata` package.

```python
from rapidata import (
    CountryFilter, LanguageFilter, UserScoreFilter, DemographicFilter,
    AgeFilter, GenderFilter, DeviceFilter, CampaignFilter, CustomFilter,
    AgeGroup, Gender, DeviceType,
    NotFilter, OrFilter, AndFilter,
)

# --- Recruitment filters on an audience: CountryFilter and LanguageFilter
#     (plus the And/Or/Not combinators). Order-only filters raise
#     NotImplementedError here; DemographicFilter belongs on .filter() (below). ---
audience.update_filters([
    CountryFilter(country_codes=["US", "CA", "GB"]),                  # 2-letter ISO codes (uppercased)
    LanguageFilter(language_codes=["en", "fr"]),                      # 2-letter ISO language codes
])

# Combine filters with logic operators
combined = OrFilter([filter1, filter2])
audience.update_filters([NotFilter(combined)])

# --- On an order: the order-only filters apply here (NOT on audiences) ---
client.order.create_classification_order(
    name="...", instruction="...", answer_options=["Yes", "No"], datapoints=["img.jpg"],
    filters=[
        UserScoreFilter(lower_bound=0.5, upper_bound=0.99),               # 0.0–1.0
        AgeFilter(age_groups=[AgeGroup.AGE_25_34, AgeGroup.AGE_35_44]),
        GenderFilter(genders=[Gender.MALE, Gender.FEMALE]),
        DeviceFilter(device_types=[DeviceType.MOBILE, DeviceType.DESKTOP]),
    ],
)

# Derive a filtered subset of a trained audience without re-onboarding labelers.
# Supported filters for .filter(): CountryFilter, LanguageFilter, DemographicFilter,
# AgeFilter, GenderFilter, DeviceFilter (plus And/Or/Not combinators).
# For DemographicFilter age, use the AgeGroup enum's .value.
filtered = base_audience.filter([
    CountryFilter(["US"]),
    LanguageFilter(["en"]),
    DemographicFilter(identifier="age", values=[AgeGroup.BETWEEN_18_29.value]),
])
job = filtered.assign_job(job_def)  # filtered is a RapidataFilteredAudience

# Combine filters with &, |, ~ operators
us_or_ca_not_fr = base_audience.filter([
    (CountryFilter(["US"]) | CountryFilter(["CA"])) & ~LanguageFilter(["fr"]),
])
```

### Filter signatures

| Filter | Signature | Works on audiences? |
|--------|-----------|---------------------|
| `CountryFilter` | `(country_codes: list[str])` | yes |
| `LanguageFilter` | `(language_codes: list[str])` | yes |
| `DemographicFilter` | `(identifier: str, values: list[str])` | `.filter()` only |
| `UserScoreFilter` | `(lower_bound: float = 0.0, upper_bound: float = 1.0, dimension: str \| None = None)` | orders only |
| `AgeFilter` | `(age_groups: list[AgeGroup])` | orders and `.filter()` |
| `GenderFilter` | `(genders: list[Gender])` | orders and `.filter()` |
| `DeviceFilter` | `(device_types: list[DeviceType])` | orders and `.filter()` |
| `CampaignFilter` | `(campaign_ids: list[str])` | orders only |
| `CustomFilter` | `(identifier: str, values: list[str])` | orders only |
| `NotFilter` | `(filter: RapidataFilter)` | both |
| `OrFilter` | `(filters: list[RapidataFilter])` | both |
| `AndFilter` | `(filters: list[RapidataFilter])` | both |

Note: recruitment filters set with `audience.update_filters(...)` are limited to `CountryFilter`, `LanguageFilter`, and the `And`/`Or`/`Not` combinators. `audience.filter(...)` (deriving a filtered audience from graduates) additionally accepts `DemographicFilter` (age/gender/occupation), `AgeFilter`, `GenderFilter`, and `DeviceFilter`. `UserScoreFilter`, `CampaignFilter`, and `CustomFilter` cannot be attached to audiences at all (they raise `NotImplementedError`) — use them as `filters=[...]` on `client.order.create_*_order(...)` instead.

## Selections (Order API only)

Selections control which rapids (validation, labeling, demographic) are presented during a session. Pass them to `client.order.create_*_order(..., selections=[...])`.

```python
from rapidata import (
    LabelingSelection, ValidationSelection, ConditionalValidationSelection,
    DemographicSelection, CappedSelection, ShufflingSelection, EffortSelection,
    RapidataRetrievalMode,
)

LabelingSelection(amount=5)                                        # 5 labeling rapids per session
LabelingSelection(amount=5, retrieval_mode=RapidataRetrievalMode.Sequential)
ValidationSelection(validation_set_id="...", amount=1)
ConditionalValidationSelection(
    validation_set_id="...",
    thresholds=[0.5, 0.8],
    chances=[1.0, 0.2],
    rapid_counts=[2, 1],
    dimensions=["global"],          # `dimension` (singular) is deprecated
)
DemographicSelection(keys=["age", "gender"], max_rapids=1)
CappedSelection(selections=[...], max_rapids=10)
ShufflingSelection(selections=[...])
EffortSelection(effort_budget=60)    # seconds of effort per session
```

`RapidataRetrievalMode` options: `Shuffled` (default), `Sequential`, `Random`.

## Results Format

### Classification Results

```json
{
  "results": {
    "globalAggregatedData": { "Cat": 15, "Dog": 8 },
    "data": [
      {
        "originalFileName": "image1.jpg",
        "aggregatedResults": { "Cat": 15, "Dog": 8 },
        "summedUserScores": { "Cat": 9.5, "Dog": 4.2 },
        "confidencePerCategory": { "Cat": 0.989, "Dog": 0.011 },
        "detailedResults": [...]
      }
    ]
  }
}
```

### Comparison Results

```json
{
  "info": { "type": "Compare", "name": "Image Comparison", "instruction": "Which image is higher quality?" },
  "results": [
    {
      "context": "A small blue book...",
      "winner_index": 1,
      "winner": "model_b.jpg",
      "assetUrls": {
        "model_a.jpg": "https://assets.rapidata.ai/<random-uuid>.jpg",
        "model_b.jpg": "https://assets.rapidata.ai/<random-uuid>.jpg"
      },
      "aggregatedResults": { "model_a.jpg": 3, "model_b.jpg": 5 },
      "aggregatedResultsRatios": { "model_a.jpg": 0.375, "model_b.jpg": 0.625 },
      "summedUserScores": { "model_a.jpg": 1.0, "model_b.jpg": 2.5 },
      "summedUserScoresRatios": { "model_a.jpg": 0.286, "model_b.jpg": 0.714 },
      "detailedResults": [
        {
          "votedFor": "model_b.jpg",
          "userDetails": {
            "country": "US", "language": "en",
            "userScores": { "global": 0.75 },
            "demographics": { "age": "25-34", "gender": "Female" }
          }
        }
      ]
    }
  ],
  "summary": { "A_wins_total": 5, "B_wins_total": 3 }
}
```

### Key Result Fields

| Field | Meaning |
|-------|---------|
| `info.type` / `info.name` / `info.instruction` | Task type (e.g. `Compare`, `Classify`), the job/order name, and the instruction shown to labelers |
| `assetUrls` | Maps each option to the Rapidata-hosted URL of the exact file shown to labelers (random-UUID filenames; not encrypted) |
| `winner` | Most-voted option (weighted by userScores) |
| `aggregatedResults` | Raw vote counts |
| `aggregatedResultsRatios` | Vote percentages |
| `summedUserScores` | Total reliability scores per option |
| `summedUserScoresRatios` | Weighted voting proportions |
| `confidencePerCategory` | Confidence level per category (with early stopping) |
| `userScore` | 0-1 value indicating individual labeler reliability |

A labeler's `demographics` may be empty when no demographic data was collected for them.

### Working with Results

```python
results = job.get_results()
df = results.to_pandas()
json_data = results.to_json()
```

Flow items have a different result shape — see the Flows section.

## Early Stopping

Two mutually exclusive strategies are available for Classification and Comparison jobs. You cannot set both on the same job.

### Confidence Stopping

Stop collecting responses once a statistical confidence threshold (weighted by labeler trust scores) is reached.

```python
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog"],
    datapoints=["pet1.jpg", "pet2.jpg"],
    responses_per_datapoint=50,   # Maximum
    confidence_threshold=0.99,    # Stop at 99% confidence
)
```

- System calculates confidence using labeler userScores
- Stops when target confidence is reached
- Saves cost by collecting fewer responses when consensus is clear
- Best for unambiguous tasks with clear correct answers
- Not recommended for subjective preference tasks

### Quorum Stopping

Stop collecting responses once a fixed number of responses agree on the same answer.

```python
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog"],
    datapoints=["pet1.jpg", "pet2.jpg"],
    responses_per_datapoint=10,   # Maximum
    quorum_threshold=7,           # Stop when 7 responses agree
)
```

A datapoint stops when:
1. `quorum_threshold` responses agree on the same answer, **OR**
2. Quorum becomes mathematically impossible (e.g. votes are too split to reach the threshold), **OR**
3. `responses_per_datapoint` total votes are collected

- Simpler than confidence stopping — based on raw vote counts, not statistics
- Good when you want predictable cost bounds with early termination
- Best for unambiguous tasks with a clear correct answer

## Cost Estimates

Both `RapidataJobDefinition` and `RapidataJob` expose an `estimated_cost` property returning a `CostEstimate` — an approximate estimate of what the job will cost to run to completion. Reading it from a job definition lets you check the cost of a run **before** assigning it to an audience. `CostEstimate` is importable from the top-level `rapidata` package.

The estimate is priced shortly after the definition/job is created, so the first read blocks briefly while polling until the estimate is available (raising `TimeoutError` if it is still not ready after a few minutes), then caches the result.

```python
job_def = client.job.create_compare_job_definition(
    name="Example Image Prompt Alignment",
    instruction="Which image matches the description better?",
    datapoints=[["midjourney.jpg", "flux.jpg"]],
    contexts=["A small blue book sitting on a large red book."],
)

estimate = job_def.estimated_cost   # blocks briefly until priced
print(f"About {estimate.estimated_cost} for {estimate.required_responses} responses")

# The same property is available once the job is running
job = audience.assign_job(job_def)
print(job.estimated_cost.estimated_cost)
```

### `CostEstimate` fields

| Field | Type | Description |
|-------|------|-------------|
| `estimated_cost` | float | Estimated total cost of running the job to completion, in your account's billing currency |
| `datapoint_count` | int | Number of datapoints the job will label |
| `required_responses` | int | Total number of responses the job collects to complete |

This is an **estimate, not the final bill**: it is based on a sample of the job's tasks scaled to the number of responses requested, so the amount actually charged can differ. Early stopping can also lower the final cost by collecting fewer responses than the maximum.

## Settings Reference

All settings inherit from `RapidataSetting` and are importable from `rapidata`.

Most settings only apply to specific task types. If you add a setting that the job's task type does not support, the SDK logs a non-fatal warning and still sends the flag — it is never dropped and no error is raised. Ranking jobs are treated as Compare for this check.

| Class | Constructor | Effect |
|-------|-------------|--------|
| `NoShuffleSetting` | `(value: bool = True)` | Disable shuffling of answer options (Likert scales) |
| `AllowNeitherBothSetting` | `(value: bool = True)` | Comparison: allow "Neither" / "Both" answers |
| `MarkdownSetting` | `(value: bool = True)` | Render markdown in text |
| `MuteVideoSetting` | `(value: bool = True)` | Start videos muted |
| `FreeTextMinimumCharactersSetting` | `(value: int)` — must be ≥ 1 | Min chars for free-text tasks. Use with caution — see note below the table |
| `FreeTextMaxCharactersSetting` | `(value: int = 1024)` — must be ≥ 1 | Max chars for free-text tasks. Use with caution — see note below the table |
| `SwapContextInstructionSetting` | `(value: bool = True)` | Swap positions of context and instruction |
| `PlayPercentageVideoSetting` | `(percentage: int = 95)` — 0–95 | Require labelers to watch N% of video |
| `OriginalLanguageOnlySetting` | `(value: bool = True)` | Skip translation; show task in original language |
| `NoMistakeOptionSetting` | `(value: bool = True)` | Hide the "mark as mistake" option |
| `DisableAutoloopSetting` | `(value: bool = True)` | Disable automatic media looping |
| `NoInstructionDisplaySetting` | `(value: bool = True)` | Hide instruction from task screen |
| `KeyboardNumericSetting` | `(value: bool = True)` | Open numeric keyboard on mobile |
| `LocateMaxPointsSetting` | `(value: int = 3)` — ≥ 1 | Locate tasks: max points per labeler |
| `LocateMinPointsSetting` | `(value: int)` | Locate tasks: min points per labeler |
| `ComparePanoramaSetting` | `(value: bool = True)` | Render comparison media as 360° panorama |
| `CompareEquirectangularSetting` | `(value: bool = True)` | Render comparison media as equirectangular VR |
| `ClassifyEquirectangularSetting` | `(value: bool = True)` | Render classification media as equirectangular 360° view |
| `CustomSetting` | `(key: str, value: str, target: "rapids" \| "campaign" = "rapids")` | Pass a custom key/value through to the backend; `target` controls whether the flag is applied at the rapid level (`"rapids"`) or campaign level (`"campaign"`) |

**Note on `FreeTextMinimumCharactersSetting` / `FreeTextMaxCharactersSetting`:** use these with caution. Free-text responses already pass through a reasonableness check by default, so tightening the bounds is usually unnecessary and will reject otherwise valid answers. Only set them when the question genuinely demands a specific length (e.g. a single word, or a full paragraph).

## Error Handling

### FailedUploadException

When datapoints fail to upload, the job is still created with the successful ones:

```python
from rapidata.rapidata_client.exceptions import FailedUploadException

try:
    job_def = client.job.create_classification_job_definition(
        name="My Job",
        instruction="...",
        answer_options=[...],
        datapoints=["valid.jpg", "missing.jpg", "valid2.jpg"],
    )
except FailedUploadException as e:
    job_def = e.job_definition          # Partially created object (or e.order for the legacy API)
    print(f"Failed: {len(e.failed_uploads)}")
    for reason, dps in e.failures_by_reason.items():
        print(f"  {reason}: {len(dps)} datapoints")
    for fu in e.detailed_failures:
        print(fu.item, fu.error_message, fu.error_type)
    # Decide whether to proceed
    if len(e.failed_uploads) <= len(datapoints) * 0.1:
        job = audience.assign_job(job_def)
```

**Properties:** `failed_uploads` (list[Datapoint] — backward-compatible), `detailed_failures` (list[FailedUpload[Datapoint]]), `failures_by_reason` (dict[str, list[Datapoint]]), `job_definition`, `order`, `dataset`.

**Recovery docs:** https://docs.rapidata.ai/3.x/error_handling/

### Jobs under review or out of funds

`assign_job` never blocks on funds: the job is always created. If its estimated cost exceeds your account balance, `assign_job` logs a warning with the estimate, your balance, and the expected shortfall — the job still runs, but may pause partway until you top up.

Some jobs don't go straight to running. A job can enter manual review (`ManualApproval`) or, once out of funds mid-run, become spend-limited (`SpendLimited`). Neither state completes on its own, so `get_results()` raises an informative error naming the state (and the review reason, when available) instead of blocking indefinitely — top up or wait for a reviewer, then call it again.

## Ranking Flows (Continuous Ranking)

Lightweight continuous ranking without full job/audience setup:

```python
# Create flow
flow = client.flow.create_ranking_flow(
    name="Image Quality Ranking",
    instruction="Which image looks better?",
    max_response_threshold=100,       # Target responses per flow item (default 100)
    min_response_threshold=50,        # Minimum acceptable responses; item is Incomplete if TTL expires below this
    # validation_set_id="...",        # Optional: validation-set id to interleave validation rapids
    # settings=[...],                 # Optional: flow-wide RapidataSettings
)

# Add items to rank
flow_item = flow.create_new_flow_batch(
    datapoints=["img1.jpg", "img2.jpg", "img3.jpg"],
    context="Generated by Model X",
    context_assets=["reference.jpg"], # Optional: 1–10 image/video/audio paths/URLs shown alongside instruction
    data_type="media",                # "media" (default) or "text"
    private_metadata=[...],           # Optional
    accept_failed_uploads=False,      # If True, proceed even if some uploads fail
    time_to_live=300,                 # Seconds until expiry (60–3600; defaults to 3600 when omitted)
)

# Get results — flow items return FlowItemResult, NOT RapidataResults
result = flow_item.get_results()      # Blocks until completed/failed/stopped/incomplete
# result.datapoints: dict[str, int]   # asset → Bradley-Terry strength estimate on Elo-scale (default start 1200);
#                                     # keyed by source URL when provided, otherwise by original filename
# result.total_votes: int             # total pairwise comparisons collected across all items

# Items sorted from best to worst
ranked = sorted(result.datapoints.items(), key=lambda item: item[1], reverse=True)

status = flow_item.get_status()       # Non-blocking check
matrix = flow_item.get_win_loss_matrix()  # Pandas DataFrame (blocks until completed)
count = flow_item.get_response_count()

# Query flow items
items = flow.get_flow_items(amount=10, page=1)

# Update a flow after creation
flow.update_config(
    instruction="New instruction",
    starting_elo=1000,
    min_responses=40,
    max_responses=120,
)

# Preheat for low-latency responses (call ~5 minutes before time-sensitive batches)
client.flow.preheat()

# Manage flows
all_flows = client.flow.find_flows(name="", amount=10, page=1)
flow = client.flow.get_flow_by_id("flow_id")
flow.delete()

# Delete other resources
job_def.delete()   # Deletes the job definition and all its revisions
job.delete()       # Deletes a running job
audience.delete()  # Deletes the audience
```

## Model Ranking Insights (MRI / Benchmarks)

Compare and rank AI models on leaderboards. Supports images, videos, audio, and text.

```python
# Create benchmark
benchmark = client.mri.create_new_benchmark(
    name="AI Art Competition",
    prompts=["A serene mountain landscape", "A futuristic city"],
    # identifiers=[...],        # Optional: stable ids for each prompt
    # prompt_assets=[...],      # Optional: reference media for each prompt
    # tags=[...],               # Optional: tags applied to the benchmark
    # description=None,         # Optional: plain-text credit for the benchmark (max 2000 characters)
)

# Add prompts later if needed (one or many, matched up by index)
benchmark.add_prompts(
    prompts=["A quiet lake at dawn"],
    # identifiers=["dawn_lake"],   # Optional: stable id per prompt
    # prompt_assets=["ref.jpg"],   # Optional: reference media per prompt
    # tags=[["landscape"]],        # Optional: list of tag lists, one per prompt
)

# Create leaderboard
leaderboard = benchmark.create_leaderboard(
    name="Realism",
    instruction="Which image is more realistic?",
    show_prompt=False,
    show_prompt_asset=False,
    inverse_ranking=False,
    # level_of_detail="high",            # "debug" | "low" | "medium" | "high" | "very high"
    # min_responses_per_matchup=5,
    # audience_id="...",                 # Optional: id string, RapidataAudience, or RapidataFilteredAudience
    # settings=[...],
    # vote_aggregation="AllVotes",       # "AllVotes" (default) or "MajorityVote" — how matchup votes are aggregated
    # benchmarkDescription="...",        # Optional: description for a newly created benchmark (max 2000 chars; ignored if benchmark already exists)
)

# Evaluate a model (creates participant, uploads media, and submits in one step)
benchmark.evaluate_model(
    name="MyModel_v2",
    media=["mountain.png", "city.png"],
    prompts=["A serene mountain landscape", "A futuristic city"],
    data_type="media",   # "media" (default) or "text"
)

# Or add a model without submitting (for more control)
participant = benchmark.add_model(
    name="MyModel_v3",
    media=["mountain_v3.png", "city_v3.png"],
    prompts=["A serene mountain landscape", "A futuristic city"],
    data_type="media",
)

# Upload additional media to the same participant
participant.upload_media(
    assets=["mountain_v3_extra.png"],
    identifiers=["A serene mountain landscape"],
    data_type="media",
)

# Submit individually or all at once
participant.run()       # Submit one participant
benchmark.run()         # Submit all unsubmitted (CREATED) participants

# Faucet — configure a participant to auto-generate samples via Replicate
participant.set_faucet(
    model_owner="stability-ai",  # Replicate model owner (e.g. "stability-ai")
    model_name="sdxl",           # Model name (e.g. "sdxl")
    model_version=None,          # Optional: pin a specific version hash
    additional_inputs={"aspect_ratio": "16:9"},  # Optional: extra model inputs (not prompt/num_outputs)
)
participant.delete_faucet()      # Remove the faucet from the participant
participant.disable()             # Exclude from evaluation and standings (reversible)
participant.enable()              # Re-enable a previously disabled participant
participant.rename("New Name")    # Rename the participant
participant.get_elo()             # Aggregated Elo across all leaderboards (None if not yet computed)
participant.delete()              # Delete participant and its uploaded media (cannot be undone)

# Sample generation — trigger a batch generation run across participants with faucets
sample_gen = benchmark.generate_samples(
    samples_per_prompt=3,         # How many samples per prompt (1–16)
    participant_ids=None,          # Optional: restrict to specific participant ids
    prompt_identifiers=None,       # Optional: restrict to specific prompt identifiers
    tags=None,                     # Optional: restrict to prompts matching any of these tags
)
# sample_gen.id                       — generation request id
# sample_gen.total_count              — total items queued
# sample_gen.skipped_participant_ids  — participants without a configured faucet

# List participants and their status (p.faucet is None if no faucet is configured)
for p in benchmark.participants:
    print(p.name, p.status, p.faucet)

# Prompts — original language and English translation (aligned by index)
print(benchmark.prompts)          # As originally provided
print(benchmark.english_prompts)  # Server-side English translations, aligned by index
print(benchmark.description)      # Optional plain-text credit (None if not set)

# Get results
standings = leaderboard.get_standings()                    # Pandas DataFrame for one leaderboard
overall = benchmark.get_overall_standings(tags=None, leaderboard_ids=None)  # Aggregated ELO across all leaderboards
matrix_lb = leaderboard.get_win_loss_matrix()              # Pairwise wins/losses for one leaderboard
matrix_bm = benchmark.get_win_loss_matrix(                 # Pairwise wins/losses across leaderboards
    tags=None, participant_ids=None, leaderboard_ids=None, use_weighted_scoring=None,
)

# Access the jobs that ran for a leaderboard (one RapidataJob per run, most recent first)
for job in leaderboard.jobs:
    job_results = job.get_results()

# Update leaderboard config live
leaderboard.name = "Realism (Updated)"
leaderboard.level_of_detail = "very high"
leaderboard.min_responses_per_matchup = 7
leaderboard.vote_aggregation = "MajorityVote"  # "AllVotes" or "MajorityVote"; only affects future runs

# Open in browser
benchmark.view()
leaderboard.view()

# Find existing benchmarks
benchmarks = client.mri.find_benchmarks(name="AI Art", amount=10)
benchmark = client.mri.get_benchmark_by_id("benchmark_id")
```

## Signals (Scheduled Labeling)

A signal runs a job definition against an audience on a repeating schedule. Each firing creates one `RapidataJob`.

### `client.signals.create_signal`

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | str | Human-readable name for the signal |
| `audience` | `RapidataAudience` \| str | Audience (or id string) the spawned jobs will target |
| `job_definition` | `RapidataJobDefinition` \| str | Job definition (or id string) each firing creates a job from |
| `interval_hours` | int | Hours between consecutive firings (minimum 1) |
| `revision_number` | int \| None | Optional: pin a specific job-definition revision; omit for "latest at fire time" |
| `is_public` | bool | If `True`, the signal is readable by every authenticated user in your org |

```python
signal = client.signals.create_signal(
    name="Daily prompt alignment",
    audience=audience,           # RapidataAudience or id string
    job_definition=job_def,      # RapidataJobDefinition or id string
    interval_hours=24,
    # revision_number=2,         # Optional: pin a revision
    # is_public=True,            # Optional: org-wide visibility
)
```

### Signal methods

| Method | Description |
|--------|-------------|
| `signal.get_jobs(page_size=10)` | List `RapidataJob` objects created by this signal |
| `signal.trigger()` | Fire one job immediately; returns right away — job created asynchronously |
| `signal.wait_for_next_job(timeout=600)` | Block until the next firing creates its job and return it |
| `signal.pause()` | Pause the scheduler (manual `trigger()` calls still fire) |
| `signal.resume()` | Resume a paused signal |
| `signal.update(name=..., interval_hours=...)` | Update the signal's name and/or cadence |
| `signal.delete()` | Delete the signal and all its runs |

### Signal manager methods

| Method | Description |
|--------|-------------|
| `client.signals.get_signal_by_id("signal_id")` | Look up a signal by id |
| `client.signals.find_signals(name="filter")` | Find signals by name |

### Signal properties

| Property | Description |
|----------|-------------|
| `id` | Unique signal id |
| `name` / `description` | Display name and optional description |
| `audience_id` | The audience each job targets |
| `job_definition_id` | The job definition each job is created from |
| `revision_number` | Pinned revision, or `None` for "latest at fire time" |
| `interval_hours` | How often the signal fires, in hours |
| `next_run_at` / `last_run_at` | Timestamps of the next and most recent firings |
| `is_paused` | Whether the scheduler is currently skipping this signal |
| `is_public` | Whether other users can discover and read it |
| `created_at` | When the signal was created |

## Configuration

```python
from rapidata import rapidata_config, logger, CompressionConfig

# Logging
rapidata_config.logging.level = "INFO"       # DEBUG, INFO, WARNING, ERROR, CRITICAL
rapidata_config.logging.log_file = "/path/to/log.txt"
rapidata_config.logging.silent_mode = False  # also suppresses terminal QR codes printed on job/order creation
rapidata_config.logging.enable_otlp = True   # OpenTelemetry tracing (auto-disabled for environments without an OTLP collector — only rapidata.ai and rabbitdata.ch have one)
rapidata_config.logging.environment = "rapidata.ai"  # API environment; derives the OTLP collector host (otlp-sdk.<environment>). Set automatically by RapidataClient from its environment

# Upload tuning
rapidata_config.upload.maxWorkers = 25        # Concurrent upload threads (warns above 200)
rapidata_config.upload.maxRetries = 3
rapidata_config.upload.cacheToDisk = True
rapidata_config.upload.cacheTimeout = 1.0
rapidata_config.upload.batchSize = 1000       # URLs per batch (100–5000)
rapidata_config.upload.batchPollInterval = 0.5
rapidata_config.upload.compression = CompressionConfig(
    enabled=True,
    quality=70,        # WebP quality 1–100
    max_dimension=1024, # Max width or height in pixels
)  # Optional: per-upload image compression override (None = server default)
rapidata_config.upload.autoShortenContext = False  # When True, auto-shorten contexts > 400 chars for the task instruction before upload

# Client-level maintenance
client.clear_all_caches()
client.reset_credentials()
```

`CompressionConfig` fields (all default `None` = defer to server):

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | `bool \| None` | Force compression on or off |
| `quality` | `int \| None` | WebP quality (1–100) |
| `max_dimension` | `int \| None` | Max width or height in pixels (≥ 1) |

Applies to single-asset uploads (`/asset/file` and `/asset/url`) and batched URL uploads.

`cacheLocation` (`~/.cache/rapidata/upload_cache`) and `cacheShards` (128) are immutable — don't try to assign them.

All config fields support environment-variable overrides with the `RAPIDATA_` prefix (e.g., `RAPIDATA_maxWorkers=10`, `RAPIDATA_DISABLE_OTLP=1`).

**Client authentication** is also resolved from environment variables: `RAPIDATA_CLIENT_ID` and `RAPIDATA_CLIENT_SECRET` are used before falling back to `~/.config/rapidata/credentials.json` and browser login. `RAPIDATA_ENVIRONMENT` overrides the API endpoint (default: `rapidata.ai`). `RAPIDATA_TOKEN_FILE` points the client at a shared access-token file (equivalent to `token_file=`). Empty values are treated as unset and fall through to the next resolution layer.

## Shared Token Files (Distributed Training)

When Rapidata is queried from a large distributed job (e.g. hundreds or thousands of GPU workers hitting a ranking flow), don't let every worker authenticate on its own: each `RapidataClient()` exchanges the client credentials for an access token that expires ~1 hour later, so all workers re-auth in the same instant and the burst gets rate-limited. Authenticate **once** and share the token via a file that all workers can read.

### `RapidataClient` authentication parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `leeway` | int | Seconds before token expiry at which the SDK refreshes/re-reads the token (default 60). A coordinator typically uses a larger value, e.g. `leeway=300`, to renew well before workers need a fresh token |
| `token_file` | str | Path to a shared token file to read the access token from; the SDK re-reads it whenever the in-memory token is within `leeway` of expiry. Also settable via the `RAPIDATA_TOKEN_FILE` env var |
| `token` | dict | An access-token dict passed directly; the SDK never re-reads it, so inject a fresh one with `client.set_token(...)` (or construct a new client) once the token expires |

### `client.maintain_token_file(path) → thread`

Writes the token file at `path` immediately, then keeps rewriting it atomically from a background thread (every 60 seconds by default), creating the directory if needed. Returns a thread-like handle; `.join()` blocks the process forever (drop it if the coordinator also does other work).

### `client.get_token() → dict`

Returns the current access token as a dict. Cheap to call at any frequency: it only contacts the auth server once the token is within `leeway` of expiry. Use it to write the shared token file yourself (write atomically, and keep the absolute `expires_at` field so workers know when to re-read).

### `client.set_token(token) → None`

The counterpart to `get_token()`: replace the token a running client authenticates with, effective from its next request, without reconstructing the client. Expects the complete token object (`access_token`, `token_type`, and an absolute `expires_at` timestamp — pass what `get_token()` returned). Together, `get_token()` and `set_token()` let you move the token over any transport (key-value store, RPC, secret manager, message queue) — a **push** system (the coordinator distributes a fresh token to every worker before the old one expires, each worker applies it with `set_token`) or a **pull** system (each worker periodically fetches the current token from your own endpoint).

```python
from rapidata import RapidataClient

# Coordinator: holds the client credentials and keeps the file fresh
coordinator = RapidataClient(leeway=300)
coordinator.maintain_token_file("/shared/rapidata_token.json").join()

# Worker: reads the shared token, never sees the client secret
client = RapidataClient(token_file="/shared/rapidata_token.json")

# Any transport: export from the coordinator, inject into a worker
token = coordinator.get_token()          # refreshes first if near expiry
worker = RapidataClient(token=token)     # bootstrap a worker from a token object
worker.set_token(coordinator.get_token())  # renew a running worker later
```

## Context Management

Datapoint contexts have a backend maximum of **400 characters** (`MAX_CONTEXT_LENGTH`). The backend rejects longer values; a warning is logged at job/order creation time.

`ContextManager` is importable from the top-level `rapidata` package and is exposed as `client.context` on every `RapidataClient` instance.

### `client.context.shorten_context(context, question) → str`

Shorten a single context for the given question. Results are cached server-side.

### `client.context.shorten_contexts(pairs) → list[str]`

Shorten a batch of `(context, question)` pairs in one request. Returns shortened contexts in the same order as `pairs`.

```python
# Single context
short = client.context.shorten_context(
    context="<a very long description ...>",
    question="Does the main character wear the right clothing?",
)

# Batch
shortened = client.context.shorten_contexts([
    (context_a, question_a),
    (context_b, question_b),
])
```

### Automatic shortening at job/order creation

Set `rapidata_config.upload.autoShortenContext = True` (default `False`) to have any context exceeding 400 characters automatically shortened against the task instruction before upload. When no instruction is available, the setting is ignored and a warning is logged instead.

```python
from rapidata import rapidata_config
rapidata_config.upload.autoShortenContext = True
```

## Human Prompting Best Practices

- **Be concise** — labelers have ~25 seconds per task
- **Positive framing** — "Which is more realistic?" not "Which is less AI-generated?"
- **Distinct options** — "Poor / Acceptable / Excellent" not "Bad / Not Good / Fine / Good / Great"
- **Single criterion per task** — "What animal is in the image?" not "Does this contain a rabbit, dog, or cat?"
- **Use `NoShuffleSetting()` for scales** — always for Likert or ordered answer options
