# The detector found nothing, and reported a clean run

I was sorting a few hundred recovered photos into folders by who was in them. Detect faces, embed them, cluster, done. It is a solved problem and the code is short.

The first run finished and printed this:

```
faces found: 46 across 40 photos
no faces   : 184
```

Forty-six faces in two hundred and twenty-four photos. That looked fine. Most of the folder was memes, screenshots and forwarded pictures of documents, so a lot of images genuinely having no face in them was exactly what I expected.

I nearly moved on. The number was plausible, and plausible is the most dangerous property a number can have.

## Checking the negatives

The only reason I looked twice was a habit rather than a suspicion: when something reports a low count, look at what it rejected, not at what it found.

So I asked what shape the 184 rejected images were:

```
     35 1200x1600 JPEG
     23 1600x1200 JPEG
     14  738x1600 PNG
      7  200x114  WEBP
```

Fifty-eight of them were portrait and landscape camera photos at ordinary phone resolutions. That is the shape of a person holding a phone and pointing it at someone. Screenshots are tall and thin, memes are small and square. Those fifty-eight were not memes.

## The number that should not have been possible

I ran the same detector over eighty of the rejected images at three confidence thresholds:

| Score threshold | Photos where a face was found |
|---|---|
| 0.85 | **0 of 80** |
| 0.60 | 28 of 80 |
| 0.35 | 40 of 80 |

Zero. Not few. At the threshold I had chosen, the detector found nothing whatsoever in eighty photographs of people, and the program reported that as a completed run with no errors.

Dropping the threshold to 0.50 took the full corpus from 46 faces to 129, and from 18 people to a set that actually matched who was in the pictures.

## Why 0.85 was wrong

0.85 is not a stupid number. It is the sort of default you pick because you want precision, and because in every example you have seen a face detector return 0.99 on a clear frontal face.

The examples are the problem. Detector benchmarks are built from photographs that are lit, framed and mostly frontal. Real photos out of a messaging app are none of those. They are dim, taken at an angle, half a face at the edge of frame, and compressed hard enough that the fine texture the model relies on has been thrown away twice by two different codecs.

The model still finds those faces. It just is not confident about them, and 0.85 asks for a confidence that real photographs rarely earn.

## The failure mode that matters

Here is what actually bothers me about this, and it is not the threshold.

**A tool that finds nothing and a tool that correctly finds nothing produce identical output.**

Both print a low number. Both exit zero. Both look like they worked. There is no error, no warning, no stack trace, nothing for a reviewer to catch. If the folder really had been all memes, 46 would have been the right answer, and I would have shipped it and never known.

This is the general shape of the problem with anything that classifies, detects, retrieves or filters. The failure does not announce itself, because absence is a legitimate result. A crash tells you something is wrong. A confidently empty result tells you nothing at all.

## What I do now

Three habits, none clever:

**Check the negatives, not the positives.** The rejected pile is where a detector's mistakes live. The accepted pile only ever shows you things that worked.

**Sweep the parameter before trusting the default.** It cost me one script and about four minutes to run three thresholds over eighty images. That is cheaper than being wrong.

**Make silent success loud.** The tool now prints what it rejected and in what shape, so the next person gets the clue I nearly missed.

I also wrote the reason into the source, immediately above the constant, because a bare number teaches nobody:

```python
# At 0.85 the detector found faces in ZERO of 80 real phone photos. At 0.6 it
# found them in 28 of the same 80. Phone pictures are noisy, badly lit and
# rarely front on, so the confidence bar has to sit low. Set it too high and the
# tool silently finds nothing while appearing to work perfectly, which is the
# worst failure mode available.
MIN_FACE_SCORE = 0.50
```

Someone will change that line one day. Now they will know what it costs.

---

The tool is at [github.com/A-e70/face-grouper](https://github.com/A-e70/face-grouper), MIT licensed. It groups photos by person and deliberately does not guess anyone's gender.
