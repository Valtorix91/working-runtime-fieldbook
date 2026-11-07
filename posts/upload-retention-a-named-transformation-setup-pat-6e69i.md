# Upload Retention: A Named Transformation Setup Pattern Explained

For a logistics marketplace generating short promotional videos from prompts, create one named transformation during setup and apply it to every cover upload through a versioned policy. Give each retained artifact an explicit expiry. The bill starts with stored originals, generated videos, listing images, cached copies, and telemetry. Count those bytes before tuning the upload script.

TL;DR: create the named transformation during deployment, require its version on every upload, and keep its identity in the resulting asset record. A Node.js upload service can enforce that contract centrally. Make retention part of the same policy: a reusable transformation name alone places no bound on stored variants or cache entries.

## What actually dominates the retained bytes?

Consider a planning example, not a measured workload. Assume 1,000 listing promotions per day, each retaining a 20 MB generated video and a 0.2 MB cover image. Use decimal units throughout. At steady daily intake, 30 days of both artifacts occupy 606 GB: 1,000 × 20.2 MB × 30. This excludes replicas, input assets, telemetry, and cache copies, so it is a deliberately incomplete capacity estimate rather than a bill forecast.

The video contributes 600 GB.

The covers contribute 6 GB.

Now suppose the business approves seven days of video retention while cover images remain available for 30 days. The corresponding estimate becomes 140 GB plus 6 GB. The change that moves the dominant term is the lifetime of the generated video. Compressing a cover image cannot make a comparable difference in this example, even though image transformations are the immediate implementation question. Actual deletion timing and billable units still depend on the storage system.

Count copies explicitly. If a second full video rendition is retained for the same seven days, it adds another 140 GB under these assumptions. A cache that holds a complete additional copy introduces a further term; do not assume every object is cached or that an expiry instantly removes stored data. Measure cache occupancy separately from origin storage. Without those distinctions, a dashboard can attribute growth to uploads while the real source is retained derivatives.

This is an artifact-lifecycle analysis. Its useful output is a decision about which evidence must survive publication, not a comparison of service prices.

## A named transformation needs an artifact budget

For the marketplace listing cover, define a versioned policy such as `listing-cover-v1`. Specify the intended dimensions, output format, and crop behavior in application configuration. Separately define the lifetime of the input image, derived cover, generated video, and diagnostic records. These lifetimes need not match. An image transformation policy should not silently become a video-generation policy merely because the artifacts belong to the same listing.

The image format decision has its own boundary. MDN's image format guide documents differences in format capabilities and browser support; it does not establish the correct dimensions, retention period, or transformation syntax for this application. Check the chosen output against the actual listing clients. An illustrative WebP cover policy is a design choice, not evidence that every client or media service accepts it.

Version the definition when its meaning changes. If an existing name acquires a new crop rule while old derivatives remain cached, the name no longer identifies one reproducible output policy. An immutable version makes an asset record interpretable during an investigation. It also creates a retention obligation: a new policy version must not imply that every historical listing receives another permanent copy.

Count the combinations.

With 1,000 assets, three requested sizes, two output formats, and two policy versions, the theoretical derivative space is 12,000 objects. That is an upper bound for the stated combination model, not an observation about a particular service. Restricting new uploads to one approved combination reduces the permitted output space, but existing objects require their own expiry or deletion process. Changing the configuration does not erase them.

## How does the upload service apply the policy every time?

Treat setup and upload enforcement as separate application responsibilities. Deployment installs the immutable policy definition and verifies that the upload service can resolve it. The upload service then supplies the required policy on every accepted upload, including retries and imports. A default in one caller is insufficient if a second caller can omit it.

The following curl calls describe a proposed internal interface for a Node.js service. They are an interface sketch, not an existing public API or a runnable provider integration. The example hostname is a placeholder; implement and test these routes before substituting a real service address. The request field names and header belong to this proposed application contract.

```bash
curl --fail-with-body --request PUT \
  'https://media.example.test/policies/listing-cover-v1' \
  --header 'Content-Type: application/json' \
  --data '{"output":{"width":960,"height":540,"format":"webp","fit":"cover"},"retention_days":30}'

curl --fail-with-body --request POST \
  'https://media.example.test/uploads' \
  --header 'X-Output-Policy: listing-cover-v1' \
  --form 'file=@./shipment-cover.jpg' \
  --form 'listing_id=shipment-demo-001'
```

In this design, the first operation creates the definition or confirms that an identical definition exists. Reusing the name with different contents must fail deployment. The second operation accepts an image only after validating the policy and recording its version with the upload. The image filename represents an input cover for a logistics listing; the prompt-generated video follows its own artifact policy. Production authentication and authorization are additional requirements, omitted from this interface sketch.

The script establishes configuration.

The service establishes enforcement.

Before rollout, test a missing policy, an unknown version, a conflicting setup definition, and a repeated upload submission. Define the application's retry identity so a retry resolves to the same logical submission instead of accumulating another permanent derivative. Neither the route name nor the HTTP method provides that upload behavior by itself. Also verify that publication waits for the required derivative to become available; accepting an upload and making a listing ready are distinct states.

Deploy the new definition before configuring writers to require it. During a transition, account for both versions in the storage estimate. A rollback may restore the earlier writer configuration, but it must not make already published assets unresolvable. Keeping a temporary compatibility window has a byte cost; state its duration before the rollout.

## Count telemetry cardinality before keeping the payload

The observability bill has another multiplication problem. Suppose a metric design permits 10,000 listing identifiers, four lifecycle stages, and three outcomes. Their Cartesian product permits 120,000 label combinations before adding any other dimensions. Actual observed series may be fewer. The calculation is still a useful warning against attaching listing identity to every metric.

Use bounded dimensions for aggregate counters: lifecycle stage, outcome, and the small set of active policy versions. Keep the listing identifier in a diagnostic record when correlation requires it. Avoid recording prompts, upload bodies, or generated media in routine telemetry; those payloads need their own access and retention decisions if a specific investigation requires them.

A separate planning assumption makes the retention cost visible. At 1,000 submissions per day, six events per submission, and 800 bytes per event, raw event payloads total 4.8 MB per day, or 144 MB over 30 days. Indexes, replicas, and metadata are outside that estimate. Compare it with the 600 GB video term before spending a week reducing log punctuation. Both terms matter, but their scale determines the order of work.

Sampling trades diagnostic coverage for fewer retained events. If policy selects one in ten successful submissions for detailed traces, some successful jobs will have no detailed history. Rare failures deserve an explicit retention rule rather than accidental exclusion by the success-sampling policy. Keep aggregate counters independent of that sampling choice so a smaller trace collection is not mistaken for fewer uploads.

## Stop retaining what the investigation does not require

The proposed boundary is concrete: expire generated videos after the approved seven-day window, retain covers for the approved 30-day window, and avoid routine payload copies in logs. These are example business decisions, not universally suitable defaults. If listings must play videos longer, the video lifetime must follow that requirement and the capacity estimate must increase.

The loss is concrete too.

Once an input or output is deleted, a policy version and a status record cannot reconstruct its exact bytes. Sparse traces may show that a job completed without explaining a visual defect reported later. Retaining the transformation identity preserves the intended rule, but it does not guarantee byte-for-byte reproduction of generated media.

Before enabling expiry, test an aged listing, a delayed complaint, and a retry after the original has expired. Decide which investigations require retained media and assign those artifacts a justified exception lifetime. The operational goal is a bounded collection whose limitations are understood: sufficient evidence to account for each submission, with deliberate gaps where exact replay no longer warrants keeping the bytes.

## References

The format capability and browser-support discussion uses the guide below. All workload sizes, policy names, interface fields, and retention windows above are explicitly illustrative design assumptions.

## Further reading

- MDN — Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
