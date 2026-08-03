# Multi-Photo Upload Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an activity owner select several photos at once in `PhotoStrip` and upload them sequentially through the existing three-phase direct-to-S3 flow, with honest per-photo progress, up-front cap refusal, and partial-success failure reporting.

**Architecture:** Client-only. The hidden file input gains `multiple`; `handleFiles` becomes: cap check (refuse whole selection before anything uploads) → per-file type/size validation (bad file recorded and skipped, never aborts the batch) → a sequential `for` loop awaiting `uploadActivityPhotoDirect` per file with per-iteration `try`/`catch`, calling `onPhotosChanged` after each successful commit. `uploadProgress: number | null` is replaced by a batch object `{ index, total, fraction }`. Failures accumulate as `{ name, reason }` and produce one grouped error toast. No endpoint, schema, or worker change.

**Tech Stack:** Next.js 16 / React 19 / TypeScript / Tailwind v4; Vitest + Testing Library (jsdom). Only `prog-strength-web` changes code; `prog-strength-docs` gets this plan + the SOW status flip.

**Repos & feature branch:** `feat/multi-photo-upload` in `prog-strength-web` and `prog-strength-docs`.

**SOW:** `prog-strength-docs/sows/multi-photo-upload.md`

## Local gate (prog-strength-web — run before every push)

```bash
cd /workspace/prog-strength-web
npm run typecheck     # tsc --noEmit — must be clean
npm run lint          # eslint — no new warnings
npm run format:check  # prettier
npm run test          # vitest run — all green
npm run build         # what CI gates merges on
```

Husky `pre-commit` runs lint-staged + `tsc --noEmit`; `commit-msg` runs commitlint (Conventional Commits). Never bypass hooks.

## Reference context (read, don't re-derive)

- **The component:** `components/activity-detail/PhotoStrip.tsx` — `handleFiles` currently takes `files?.[0]` and discards the rest; `uploading: boolean` + `uploadProgress: number | null` drive the button label (`Uploading 62%`). The `processing` placeholder and the poll-until-none-processing effect already handle several in-flight photos — do not touch them.
- **The upload flow:** `lib/api.ts` → `uploadActivityPhotoDirect(token, activityId, file, opts?)` (reserve → PUT to S3 → commit). `opts.onProgress: (fraction: number) => void` reports the S3 PUT's progress for one file. Do not modify `lib/api.ts`.
- **Existing tests:** `components/activity-detail/PhotoStrip.test.tsx` — note its module mock still stubs the retired `uploadActivityPhoto`; the component now imports `uploadActivityPhotoDirect`, so the mock must be updated.
- **The cap:** server config `photos.max_per_activity = 10` (see `sows/activity-photos.md`). The client mirrors it as a constant, same as the existing `MAX_PHOTO_BYTES` / `ALLOWED_PHOTO_TYPES` guards — UX-only; the API stays authoritative (409 `photo_limit_reached`).
- **Design system:** no new colors/tokens; the button keeps its existing `--accent` text-link treatment. Toasts via the existing `useToast()`.

---

### Task 1: `PhotoStrip` multi-select batch upload

**Files:**
- Modify: `components/activity-detail/PhotoStrip.tsx`
- Test: `components/activity-detail/PhotoStrip.test.tsx`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-web
git checkout -b feat/multi-photo-upload
```

- [ ] **Step 2: Rewrite the test file's mock plumbing and add the batch tests (failing first)**

In `components/activity-detail/PhotoStrip.test.tsx`:

1. Replace the hoisted `uploadActivityPhotoMock` with `uploadActivityPhotoDirectMock`, and update the `vi.mock("@/lib/api", …)` factory to export `uploadActivityPhotoDirect` instead of `uploadActivityPhoto` (keep the other three mocks unchanged):

```tsx
const uploadActivityPhotoDirectMock = vi.hoisted(() => vi.fn());
const updateActivityPhotoCaptionMock = vi.hoisted(() => vi.fn());
const reorderActivityPhotosMock = vi.hoisted(() => vi.fn());
const deleteActivityPhotoMock = vi.hoisted(() => vi.fn());
const errorToastMock = vi.hoisted(() => vi.fn());

vi.mock("@/lib/api", () => ({
  uploadActivityPhotoDirect: uploadActivityPhotoDirectMock,
  updateActivityPhotoCaption: updateActivityPhotoCaptionMock,
  reorderActivityPhotos: reorderActivityPhotosMock,
  deleteActivityPhoto: deleteActivityPhotoMock,
}));
```

2. Add two helpers next to the existing `photo()` helper:

```tsx
/** A File with a controllable type and size — jsdom Files are always tiny, so size is defined explicitly. */
function makeFile(name: string, type = "image/jpeg", size = 1024): File {
  const file = new File(["x"], name, { type });
  Object.defineProperty(file, "size", { value: size });
  return file;
}

function selectFiles(files: File[]) {
  fireEvent.change(screen.getByLabelText("Add photos"), { target: { files } });
}
```

3. Add a new `describe("PhotoStrip batch upload", …)` block with these tests (SOW's Testing section, one test each):

```tsx
describe("PhotoStrip batch upload", () => {
  it("uploads a batch of three in selection order, refreshing after each commit", async () => {
    uploadActivityPhotoDirectMock.mockResolvedValue(photo("new", 0));
    const onPhotosChanged = vi.fn();
    renderStrip({ photos: [], onPhotosChanged });

    selectFiles([makeFile("a.jpg"), makeFile("b.jpg"), makeFile("c.jpg")]);

    await waitFor(() => expect(uploadActivityPhotoDirectMock).toHaveBeenCalledTimes(3));
    const uploadedNames = uploadActivityPhotoDirectMock.mock.calls.map(
      (call) => (call[2] as File).name,
    );
    expect(uploadedNames).toEqual(["a.jpg", "b.jpg", "c.jpg"]);
    // One refresh per successful commit — photos appear as they land.
    expect(onPhotosChanged).toHaveBeenCalledTimes(3);
    expect(errorToastMock).not.toHaveBeenCalled();
  });

  it("a mid-batch failure keeps earlier successes and still attempts later files", async () => {
    uploadActivityPhotoDirectMock
      .mockResolvedValueOnce(photo("n1", 0))
      .mockRejectedValueOnce(new Error("Upload failed before the server responded"))
      .mockResolvedValueOnce(photo("n2", 1));
    const onPhotosChanged = vi.fn();
    renderStrip({ photos: [], onPhotosChanged });

    selectFiles([makeFile("a.jpg"), makeFile("b.jpg"), makeFile("c.jpg")]);

    // The failure does not abort the loop: all three are attempted.
    await waitFor(() => expect(uploadActivityPhotoDirectMock).toHaveBeenCalledTimes(3));
    expect(onPhotosChanged).toHaveBeenCalledTimes(2);
    await waitFor(() =>
      expect(errorToastMock).toHaveBeenCalledWith("1 photo failed — 1 upload error."),
    );
  });

  it("an over-cap selection uploads nothing and names the remaining slots", async () => {
    // Strip already holds 3 photos; cap is 10 → 7 slots remain.
    renderStrip();

    selectFiles(Array.from({ length: 8 }, (_, i) => makeFile(`f${i}.jpg`)));

    await waitFor(() =>
      expect(errorToastMock).toHaveBeenCalledWith(
        "You can add 7 more photos — you selected 8.",
      ),
    );
    expect(uploadActivityPhotoDirectMock).not.toHaveBeenCalled();
  });

  it("reports the correct index/total as the batch advances", async () => {
    let resolveUpload!: (value: unknown) => void;
    uploadActivityPhotoDirectMock.mockImplementation(
      (_token, _activityId, _file, opts: { onProgress: (fraction: number) => void }) =>
        new Promise((resolve) => {
          resolveUpload = resolve;
          opts.onProgress(0.62);
        }),
    );
    renderStrip({ photos: [] });

    selectFiles([makeFile("a.jpg"), makeFile("b.jpg")]);

    // Mid-batch the Add button doubles as the progress readout and is
    // disabled — a second selection during a batch would scramble the count.
    const uploading = await screen.findByRole("button", { name: "Uploading 1 of 2 — 62%" });
    expect(uploading).toBeDisabled();
    resolveUpload(photo("n1", 0));
    await screen.findByRole("button", { name: "Uploading 2 of 2 — 62%" });
    resolveUpload(photo("n2", 1));
    // Batch done: the button reverts to its label and is clickable again.
    expect(await screen.findByRole("button", { name: "+ Add photos" })).toBeEnabled();
  });

  it("a full strip refuses any selection with the at-cap message", async () => {
    // Already at the cap of 10 — the toast names the cap instead of offering
    // "0 more" slots.
    renderStrip({ photos: Array.from({ length: 10 }, (_, i) => photo(`p${i}`, i)) });

    selectFiles([makeFile("a.jpg")]);

    await waitFor(() =>
      expect(errorToastMock).toHaveBeenCalledWith(
        "This activity already has the maximum of 10 photos.",
      ),
    );
    expect(uploadActivityPhotoDirectMock).not.toHaveBeenCalled();
  });

  it("skips a file rejected on type without aborting the batch", async () => {
    uploadActivityPhotoDirectMock.mockResolvedValue(photo("new", 0));
    renderStrip({ photos: [] });

    selectFiles([makeFile("bad.heic", "image/heic"), makeFile("good.jpg")]);

    await waitFor(() => expect(uploadActivityPhotoDirectMock).toHaveBeenCalledTimes(1));
    expect((uploadActivityPhotoDirectMock.mock.calls[0][2] as File).name).toBe("good.jpg");
    await waitFor(() =>
      expect(errorToastMock).toHaveBeenCalledWith("1 photo failed — 1 unsupported format."),
    );
  });

  it("groups mixed failures into one toast", async () => {
    uploadActivityPhotoDirectMock
      .mockRejectedValueOnce(new Error("network down"))
      .mockRejectedValueOnce(new Error("server sad"));
    renderStrip({ photos: [] });

    selectFiles([
      makeFile("bad.heic", "image/heic"),
      makeFile("huge.jpg", "image/jpeg", 13 * 1024 * 1024),
      makeFile("a.jpg"),
      makeFile("b.jpg"),
    ]);

    await waitFor(() => expect(uploadActivityPhotoDirectMock).toHaveBeenCalledTimes(2));
    await waitFor(() =>
      expect(errorToastMock).toHaveBeenCalledWith(
        "4 photos failed — 1 unsupported format, 1 file over 12 MB, 2 upload errors.",
      ),
    );
    expect(errorToastMock).toHaveBeenCalledTimes(1);
  });
});
```

4. The two existing owner-affordance tests query `/add photo/i` — this regex still matches the new "+ Add photos" label, so leave them as-is. The input's `aria-label` becomes `"Add photos"` (the `selectFiles` helper depends on it).

- [ ] **Step 3: Run the new tests to verify they fail**

```bash
npx vitest run components/activity-detail/PhotoStrip.test.tsx
```

Expected: the seven new batch tests FAIL (component still uploads only `files[0]`, mocks changed); pre-existing tests pass.

- [ ] **Step 4: Implement the batch upload in `PhotoStrip.tsx`**

All changes in `components/activity-detail/PhotoStrip.tsx`.

1. Extend the guard constants and add the failure vocabulary (replaces the existing two-constant block at the top):

```tsx
// Client-side upload guards — UX only; the API is authoritative on size,
// content type, and the per-activity cap. Catching here gives a snappier
// rejection than a round-trip. The cap mirrors the server's
// `photos.max_per_activity`.
const MAX_PHOTO_BYTES = 12 * 1024 * 1024;
const ALLOWED_PHOTO_TYPES = new Set(["image/jpeg", "image/png", "image/webp"]);
const MAX_PHOTOS_PER_ACTIVITY = 10;

// Why a file did not attach, grouped for the batch's single failure toast.
// "unsupported format" and "file over 12 MB" are rejections — the user's move
// is to pick a different file. "upload error" is a network/server failure —
// retrying the same file is reasonable. The label pair is (singular, plural)
// because the toast counts each class.
const FAILURE_LABELS = {
  unsupportedType: ["unsupported format", "unsupported formats"],
  tooLarge: ["file over 12 MB", "files over 12 MB"],
  uploadError: ["upload error", "upload errors"],
} as const;
type FailureReason = keyof typeof FAILURE_LABELS;
type UploadFailure = { name: string; reason: FailureReason };

/** One grouped message: "3 photos failed — 1 unsupported format, 2 upload errors." */
function describeFailures(failures: UploadFailure[]): string {
  const parts = (Object.keys(FAILURE_LABELS) as FailureReason[])
    .map((reason) => ({ reason, count: failures.filter((f) => f.reason === reason).length }))
    .filter(({ count }) => count > 0)
    .map(({ reason, count }) => `${count} ${FAILURE_LABELS[reason][count === 1 ? 0 : 1]}`);
  return `${failures.length} photo${failures.length === 1 ? "" : "s"} failed — ${parts.join(", ")}.`;
}
```

2. Replace the `uploading` / `uploadProgress` state pair with one batch object (the presence of the object is the "uploading" flag):

```tsx
// Non-null while a batch is in flight: which photo is uploading (1-based),
// how many the batch holds, and the current file's S3 PUT fraction (0..1).
// "Uploading 3 of 8 — 62%" is literally true because uploads are sequential.
const [batchProgress, setBatchProgress] = useState<{
  index: number;
  total: number;
  fraction: number;
} | null>(null);
```

3. Replace `handleFiles` entirely:

```tsx
const handleFiles = async (fileList: FileList | null) => {
  const files = fileList ? Array.from(fileList) : [];
  if (files.length === 0) return;

  // Cap check runs before anything uploads. Refusing the whole selection up
  // front beats filling the remaining slots in selection order, which would
  // silently pick winners.
  const remaining = MAX_PHOTOS_PER_ACTIVITY - photos.length;
  if (files.length > remaining) {
    // A full strip gets its own wording — "You can add 0 more photos" reads
    // like a riddle when the real answer is "the activity is full".
    toast.error(
      remaining <= 0
        ? `This activity already has the maximum of ${MAX_PHOTOS_PER_ACTIVITY} photos.`
        : `You can add ${remaining} more photo${remaining === 1 ? "" : "s"} — you selected ${files.length}.`,
    );
    return;
  }

  const token = getToken();
  if (!token) return;

  // Per-file validation: a bad file is recorded and skipped — one HEIC in a
  // selection of eight must not cost the other seven.
  const failures: UploadFailure[] = [];
  const valid: File[] = [];
  for (const file of files) {
    if (!ALLOWED_PHOTO_TYPES.has(file.type)) {
      failures.push({ name: file.name, reason: "unsupportedType" });
    } else if (file.size > MAX_PHOTO_BYTES) {
      failures.push({ name: file.name, reason: "tooLarge" });
    } else {
      valid.push(file);
    }
  }

  // Sequential on purpose: parallel uploads contend for the same upstream,
  // let position assignment interleave, and turn "photo 3 of 8" into a
  // fiction. Each iteration owns its try/catch so one failure cannot abort
  // the rest — three phases per file: reserve, PUT straight to S3, commit.
  for (const [i, file] of valid.entries()) {
    setBatchProgress({ index: i + 1, total: valid.length, fraction: 0 });
    try {
      await uploadActivityPhotoDirect(token, activityId, file, {
        onProgress: (fraction) => setBatchProgress({ index: i + 1, total: valid.length, fraction }),
      });
      // Refresh after every commit, not once at the end — each photo appears
      // in the strip (as a processing placeholder) the moment it lands.
      onPhotosChanged();
    } catch (err) {
      // The grouped toast stays terse on purpose, so the specific error
      // (status code, server message) lands in the console for debugging.
      console.error("photo upload failed", file.name, err);
      failures.push({ name: file.name, reason: "uploadError" });
    }
  }
  setBatchProgress(null);

  if (failures.length > 0) {
    toast.error(describeFailures(failures));
  }
};
```

4. Update the Add button and file input in the JSX (`multiple`, plural labels, batch label):

```tsx
<button
  type="button"
  onClick={() => fileInputRef.current?.click()}
  disabled={batchProgress !== null}
  className="text-xs text-[var(--accent)] hover:underline disabled:opacity-40"
>
  {batchProgress
    ? `Uploading ${batchProgress.index} of ${batchProgress.total} — ${Math.round(batchProgress.fraction * 100)}%`
    : "+ Add photos"}
</button>
<input
  ref={fileInputRef}
  type="file"
  accept="image/*"
  multiple
  aria-label="Add photos"
  className="hidden"
  onChange={(e) => {
    void handleFiles(e.target.files);
    // Reset so re-selecting the same file re-fires onChange.
    e.target.value = "";
  }}
/>
```

5. Update the component docblock's owner-affordances bullet to describe the batch behavior (multi-select input, up-front cap refusal, per-file validation, sequential three-phase uploads with per-photo progress, grouped failure toast, `uploadActivityPhotoDirect` not `uploadActivityPhoto`).

- [ ] **Step 5: Run the component tests to verify they pass**

```bash
npx vitest run components/activity-detail/PhotoStrip.test.tsx
```

Expected: ALL tests pass (existing + seven new; 15 in the file).

- [ ] **Step 6: Run the full local gate**

```bash
npm run typecheck && npm run lint && npm run format:check && npm run test && npm run build
```

Expected: all clean. Fix anything that fails (Prettier: `npm run format` then re-check).

- [ ] **Step 7: Commit**

```bash
git add components/activity-detail/PhotoStrip.tsx components/activity-detail/PhotoStrip.test.tsx
git commit -m "feat(photos): select and upload several photos at once"
```

Expected: Husky pre-commit (lint-staged + tsc) and commitlint pass.

---

### Task 2: Docs — commit the plan and flip the SOW to shipped

**Files:**
- Create (already authored; commit it): `plans/2026-08-03-multi-photo-upload.md` in `prog-strength-docs`
- Modify: `sows/multi-photo-upload.md` in `prog-strength-docs`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-docs
git checkout -b feat/multi-photo-upload
```

- [ ] **Step 2: Flip the SOW status**

In `sows/multi-photo-upload.md`:
- Frontmatter: `status: draft` → `status: shipped`.
- Body header line: `**Status**: Draft · **Last updated**: 2026-08-03` → `**Status**: Shipped · **Last updated**: 2026-08-03`.

No other edits to the SOW.

- [ ] **Step 3: Commit**

```bash
git add plans/2026-08-03-multi-photo-upload.md sows/multi-photo-upload.md
git commit -m "docs: mark multi-photo-upload as shipped"
```

Expected: commit lands on `feat/multi-photo-upload`; `git status` clean.
