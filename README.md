# tenement-omni-explorer assets

The point clouds and Gaussian splats for
[tenement-omni-explorer](https://github.com/ivanmkc/tenement-omni-explorer). Data only —
no code, no build step. The viewer fetches these files cross-origin, on demand, a byte
range at a time.

## Why they are not in the main repository

They used to be, and it cost more than it was worth. Thirty clouds are 455 MB in a working
tree, and carrying every revision of them had grown the main repository's pack to 1.04 GiB
— a cost paid by every clone, including the ones that only wanted the explorable. They
were taken out of git entirely and history was rewritten; the pack is now about 96 MB.

## Why they are not bundled into the Pages site either

The viewer is published to GitHub Pages, and for a while the clouds were copied into that
publish. That is a worse version of the same problem: the reader downloads the site, the
site contains 827 MB of reconstruction, and almost all of it is for a cloud they will
never open. Only one cloud is on screen at a time.

So the site ships the manifest — a 20 KB index of what exists, which cloud came from which
reconstruction, how many points each has — and nothing else. The bytes come from here,
when and if somebody asks to see them.

## Why a separate public repository, rather than a bucket

A bucket was the obvious answer and it is not available. `gs://ivanmkc-recon-viewer` holds
these same files and is the source of truth for producing them, but it cannot be read from
a browser: `constraints/storage.publicAccessPrevention` is enforced org-wide, so no object
can be granted to `allUsers`, and `iam.allowedPolicyMemberDomains` confines members to two
customer IDs. A Cloud Run proxy in front of it fails on the same constraint. GitHub
Releases serve assets without an `access-control-allow-origin` header, so a browser refuses
them. Git LFS serves nothing readable from a private repository and Pages never smudges LFS
pointers regardless.

What is left, and what this is, is an ordinary public repository. `raw.githubusercontent.com`
returns `access-control-allow-origin: *` and honours `Range` with a `206` and an exact
`content-range`, which is the whole requirement.

## Why the files are shuffled

Both formats are read as a prefix. The viewer's detail slider does not choose how many
points to *draw* out of a full download — it chooses how many to *download*. Asking for 10%
transfers 10% of the bytes.

That is only honest if a prefix is a fair sample of the scene, which is a property of how
the files are written, not of how they are read. These clouds come out of their producers
in the order the producer emitted them: raster order for a back-projected depth map,
training order for a Gaussian world. A prefix of raster order is a horizontal band across
the top of the panorama — drag the slider down and the room loses its floor. So every file
here was shuffled with a fixed seed before it was packed, whether or not anything was
dropped from it, and nothing re-sorts them afterwards.

## Formats

`.bin` is `RCLD`: a 4-byte magic, a `uint32` point count, then *all* positions
(`3 × float32`) and then *all* colours (`3 × uint8`). It is blocked rather than interleaved,
so reading the first N points takes two range requests — one per block — and they have to
agree on N or the colours belong to different points than the positions.

`.splat` is the interleaved 32-byte record used by antimatter15's viewer, behind a 12-byte
header (magic, version, count). Position `3 × float32`, scale `3 × float32`, colour
`4 × uint8`, rotation `4 × uint8`. Interleaved means one range request is enough.

In both cases the count in the header describes the whole file, not the part that arrived.
A ranged reader has to trust the bytes it got rather than the header, or it walks off the
end of the buffer.
