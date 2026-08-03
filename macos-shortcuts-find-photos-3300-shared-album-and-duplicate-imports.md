# macOS Shortcuts `Find Photos` Fails with `PHPhotosErrorDomain 3300`

## TL;DR

Several related failures appeared while building a macOS Shortcut that finds the 150 largest photos and videos in a large Photos library:

1. A whole-library `Find Photos` / sort operation could not reliably finish. Very small limits worked. That did not prove a memory overflow by itself, but it justified a bounded design that never depends on materializing the entire library in one Shortcut run.
2. Even after partitioning by year and month, November 2018 consistently failed inside `Find Photos` with:

```text
The operation couldn’t be completed.
(PHPhotosErrorDomain error 3300.)
```

The November 2018 failure was caused by two photos that existed in both scopes:

- normal full-resolution HEIC assets in the main library; and
- lower-resolution JPEG proxy assets in an iCloud Shared Album, anonymized here as `Shared Album A`.

The originals were healthy: both could be exported and manually added to a normal album. The working fix was to exclude the shared album **inside every affected `Find Photos` action**:

```text
Album is not Shared Album A
```

The cloned video workflow failed in June 2025 for the same reason. Four matching video results came from another shared album (`Shared Album B`). A third shared album (`Shared Album C`) contained additional video proxies in July 2025. The video query therefore excluded all three reproducing shared albums.

A separate duplicate incident came from using a save/import action as though it were an idempotent album-membership action. Re-running it against an album that already contained the same images created new library items. The stable practice is to write Photos assets to a fresh empty output album, or remove the previous album membership before rebuilding it, and never use `Save File` to represent “add this existing Photos asset to an album.”

## Verified Case

Verified on 2026-08-03 with macOS Photos and Shortcuts.

The workflow contained shortcuts equivalent to:

```text
Build Year Candidates Photo Master
Build Year Candidates Photo
```

The intended outputs were separate photo and video albums containing the largest items by file size.

## Symptoms

### Whole-library query pressure

- An unrestricted `Find Photos` action stalled or failed.
- Setting the limit to five completed quickly.
- Photos and Shortcuts already had full Photos access.
- “Allow Sharing Large Amounts of Data” was enabled.
- Increasing permissions did not change the failure.

### Deterministic 2018 failure

- Processing 2006 through 2017 worked.
- Processing 2018 failed consistently.
- Running 2018 by itself still failed, ruling out only cumulative state from earlier years.
- Dividing 2018 by month isolated the failure to November 2018.
- Dividing the November results isolated two visual photos.
- Both photos exported successfully.
- Both main-library originals could be manually added to a new ordinary album.
- `Find Photos` still returned `PHPhotosErrorDomain error 3300` until `Shared Album A` was excluded.

### Deterministic June 2025 video failure

- The cloned video workflow processed earlier years and stopped in June 2025 with the same error 3300.
- Read-only Photos-library inspection found four June 2025 videos from `Shared Album B`.
- Excluding `Shared Album B` made the failing video query pass.
- `Shared Album C` contained 23 additional video proxies dated in July 2025, so it was excluded before continuing the full scan.

### Duplicate library items

- A save action targeting the result album looked like it was merely adding membership.
- In reality, it saved/imported another file into Photos.
- Re-running the workflow when the target album already contained the same content created duplicates.

## Scalability Constraint: Avoid Unbounded Intermediate Results

The original Shortcut asked Photos for a very large result set and then performed file-size work over that set. Small limits completed while the unrestricted form did not.

That observation was consistent with resource pressure, but it was not enough to prove an allocation overflow: a small limit can also avoid a problematic asset that appears later in the result set. The later Shared Album finding supplied that alternative explanation. The bounded Top-N design remains preferable because it removes both unnecessary intermediate state and dependence on a single all-library fetch.

### Exact bounded Top-N reduction

For a desired global Top 150, retain the Top 150 from every partition, merge those candidates, and take the Top 150 again.

For example:

```text
each month -> largest 150
12 monthly candidate lists -> largest 150 for the year
all yearly candidate lists -> largest 150 overall
```

This remains exact. An item ranked below 150 inside its own month cannot be in the global Top 150, because at least 150 items from the same month are already larger.

Do not use a candidate limit of five for the final workflow. Five is useful for debugging, but it cannot guarantee the correct global Top 150.

### Correct cache compaction

The working year-boundary merge used:

```text
Find Photos where Any are true:
  Album is Current Candidate Cache
  Album is Previous Candidate Cache
```

`Any` is essential because the workflow needs the union. The earlier `All` form returned only photos present in both albums, so most candidates were never included in the removal input and accumulated into the thousands.

In the verified Shortcut, `Find Photos` produced a `Photos` value and the file-size filter produced a `Files` value. The two `Remove Photos from Album` actions therefore received the earlier compatible `Photos` union. Removing that same union from each cache is valid: items absent from one album are harmless no-ops, while every item present in either album is removed from its owning cache.

After clearing both caches, only the filtered Top 150 is written back to the current cache. The master-level startup cleanup must also remove the current-cache results from the current cache, not accidentally target the final output album.

### Recommended date partition

For each month:

```text
Start Date = first day of the month
End Date   = Start Date adjusted by +1 month
```

Then query:

```text
Find Photos where all are true:
  Media Type is Image
  Date Taken is between Start Date and End Date
  Album is not Shared Album A
```

Sort/filter the result by:

```text
File Size
Biggest First
Limit 150
```

Run the equivalent pipeline separately for videos.

## Confirmed Root Cause: A Shared-Album Proxy Pair Broke `Find Photos`

The two November 2018 scenes each had two distinct Photos records:

- a 4032×3024 HEIC in the normal main library; and
- a 2048×1536 JPEG in `Shared Album A`.

The paired records represented the same visual image and had capture timestamps less than one second apart. The Shared Album records were normal readable proxy assets, not corrupted JPEG files.

The important distinction was:

- **Main-library originals:** exportable and manually addable to a normal album.
- **Shared Album proxies:** readable, but caused the date-scoped `Find Photos` operation to fail when included in this query.

Other Shared Album photos did not automatically fail. Shared Album membership alone was not the sufficient condition. The reproducing condition was this specific query encountering the overlapping main-library and shared-proxy records.

### Final working fix

Add the exclusion to the failing `Find Photos` predicate itself:

```text
Album is not Shared Album A
```

Do not place this only in a later `Filter Photos` action. If `Find Photos` fails before producing output, a downstream filter never receives anything to filter.

After adding the predicate, November 2018 completed successfully.

For the verified video workflow, the query excluded the three shared albums that contained reproducing video proxies:

```text
Album is not Shared Album A
Album is not Shared Album B
Album is not Shared Album C
```

These exclusions belong under `All`, alongside media type and date range. Each exclusion must be true.

## Duplicate Root Cause: Saving Files Is Not Album Membership

Photos assets and exported files are different types of objects in Shortcuts.

An action that saves a file into Photos can create a new Photos library asset. It is not equivalent to adding an existing `Photo` object to an album. Repeating that operation is not idempotent and can create duplicates.

An album action whose input pill displays `Files` is still crossing this import boundary. Clearing or recreating the destination album prevented the observed rerun duplication, but a membership-only design should preserve Photos asset identity instead of relying on file re-import.

### Stable album rebuild pattern

Use this lifecycle:

1. Create a new empty output album, or remove the previous run's items from the existing output album.
2. Pass Photos asset objects to a Photos album action.
3. Do not use `Save File` as an album-membership operation.
4. Run the workflow.
5. Verify the output album before removing any prior known-good album.

If a repeated run produces duplicates, stop and inspect the action's input and output types. Do not keep rerunning it against the same album.

## Troubleshooting Method That Located the Failure

### 1. Identify the exact red action

The parent `Run Shortcut` action only reports that its child failed. Open the child shortcut and confirm whether the red action is:

- `Find Photos`;
- file-size filtering;
- album creation; or
- a save/import action.

In this incident, the decisive error occurred in `Find Photos`.

### 2. Partition time instead of guessing

Reduce the date range in this order:

```text
all years -> one year -> half-year -> month
```

The failure narrowed to:

```text
2018 -> November 2018
```

### 3. Isolate individual assets

Within the failing month, reduce the result limit or split the result list until the exact visual assets are known.

### 4. Separate file health from PhotoKit query health

For each suspect photo:

- export it;
- manually add the main-library original to a new normal album;
- test `Find Photos` without any album-writing action.

Interpretation:

- Export fails: investigate unavailable or damaged resources.
- Manual album addition fails: investigate the main Photos asset/catalog relationship.
- Both succeed but `Find Photos` fails: investigate query scope and proxy/shared records.

In this case, export and manual album addition both succeeded.

### 5. Exclude the reproducing scope at query time

Adding `Album is not Shared Album A` made the failing photo month pass. Excluding `Shared Album B` made the failing video month pass. These results confirmed the relevant query boundary without deleting or re-importing the original assets.

## Misleading Hypotheses

### Empty years or months

An empty partition should return an empty result. It does not explain a deterministic `PHPhotosErrorDomain 3300` tied to two specific records.

### Photos permissions

Full Photos access and large-data sharing were already enabled. Re-granting them did not address the failing query.

### A damaged JPEG

Both derivatives decoded correctly, and both originals exported. The visible files were not corrupt.

### Missing local original

One original initially needed downloading, but the error remained after it was successfully exported and present locally. Local availability was not the final cause.

### All Shared Albums are broken

The library contained many other Shared Album assets. The fix only needed to exclude the reproducing album. Do not turn this into a rule that all shared content must be removed.

### Filtering after `Find Photos`

A downstream filter cannot recover from a `Find Photos` action that already failed. Put the exclusion inside the query.

## Final Working Design

```text
For each year:
  For each month:
    Find Photos by media type and date
      AND Album is not each reproducing Shared Album
    Sort by File Size, Biggest First
    Keep 150 monthly candidates

  Merge monthly candidates
  Sort by File Size, Biggest First
  Keep 150 yearly candidates

Merge yearly candidates
Sort by File Size, Biggest First
Keep final 150

Write Photos assets to a fresh empty output album
```

The video workflow uses the same bounded reduction with `Media Type is Video`, plus exclusions for every shared album that contains reproducing video proxies.

## Final Conclusions

The incident contained two confirmed mechanisms and one design constraint:

1. **Scalability constraint:** the all-library form was unreliable, but memory overflow was not directly proven. Hierarchical Top-N reduction avoids the risk without changing the answer.
2. **PhotoKit query scope:** shared-album proxy records caused deterministic photo and video failures. Excluding the reproducing shared albums inside `Find Photos` removed error 3300.
3. **Type/action mismatch:** a save/import action was used where album membership was intended, so reruns created duplicate library items.

The stable solution combines bounded Top-N reduction, in-query Shared Album exclusions, `Any`-based cache union and clearing, and fresh/cleared album outputs that preserve Photos asset identity instead of repeatedly importing saved files.
