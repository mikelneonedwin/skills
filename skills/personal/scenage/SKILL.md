---
name: scenage
description: Arranges and checks movie and TV show downloads by automatically moving, renaming, and finding missing episodes or subtitles.
---

# Scenage (Media Organizer)

This skill empowers you to act as an intelligent media organizer. When the user asks you to organize media, arrange downloads, or check for missing episodes, you should execute the following behaviors using your own tools and capabilities (e.g., by dynamically writing your own bash/python scripts).

## 1. Detect & Parse Filenames
Scan the given folder for video files (`.mp4`, `.mkv`, `.avi`, etc.) and subtitle files (`.srt`, `.ass`, `.vtt`, `.sub`).

When parsing filenames:
- **Clean the title**: Remove quality tags (like `1080p`, `720p`), language tags (`English`, `French`), and replace dots/underscores with spaces. Capitalize each word.
- **TV Series Detection**: Look for patterns like `S01E02`, `S1E2`, or `s01e02`. Extract the Series Title, Season number, and Episode number.
- **Movie Fallback**: If it doesn't match a series pattern, treat it as a Movie.

## 2. Arrange (Move & Rename)
When asked to arrange files:
1. **For Movies**: Move them to `[Output Folder]/Movies/[Movie Title].[ext]`.
2. **For TV Series**: Move them to `[Output Folder]/[Series Title]/S[Season]/[Series Title] S[Season]E[Episode].[ext]`. (Use 2-digit zero-padding for season and episode by default, e.g., `S01E02`).
3. **Subtitles**: Rename subtitles to exactly match their corresponding video filenames, keeping their subtitle extension.
   - **Handling Sequential Duplicate Subtitles**: Often, bulk-downloaded subtitles share identical names and get auto-renamed by the OS (e.g., `subtitle.srt`, `subtitle (1).srt`, `subtitle (2).srt`). When arranging, map these sequentially to the sorted video episodes. Line them up such that the base file (`subtitle.srt`) maps to the first episode, `(1)` maps to the second episode, `(2)` maps to the third, and so on until `(n)`.
4. **Collision Handling**: If a file already exists at the destination, append a counter like `(1)` to the filename.

## 3. Check (Consistency & Missing Files)
When asked to check media consistency:
1. Group all detected files by Series and then by Season.
2. Find the maximum episode number for each season based on the files present.
3. Identify **missing episodes**: e.g., if you have E01 and E03, report E02 as missing.
4. Identify **missing subtitles**: if a video exists but no matching subtitle exists.
5. Identify **missing seasons**: if Season 1 and 3 exist, Season 2 is missing.
6. Print a clean, readable report grouping these issues by series.
