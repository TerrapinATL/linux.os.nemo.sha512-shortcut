### linux-audio-nemo-actions

**Version: v1** — First version. Merges the SHA512 and ReplayGain Nemo actions into a single guide, and adds tag-verification and tag-writing actions.

---

01. Introduction

---

This guide installs a set of Nemo right-click actions for moOde-aware music libraries on Linux Mint. It brings together the SHA512 checksum verification actions and the ReplayGain actions into one place, as part of the **moOde Library Integrity Suite**.

Unlike the whole-library guides in this suite (moOde Cleanup and SHA512 Library), these Nemo actions are designed for quick, **Artist- or Album-specific** work. They are invoked by right-clicking on a file or folder in the Nemo file manager and run instantly, without loading a full workflow.

The actions installed here are:

* Verify ALBUM SHA512 Checksums — verifies the individual track files in an album folder.
* Verify ARTIST SHA512 Checksums — verifies each album directory inside an artist folder.
* Show ReplayGain — displays the current ReplayGain tags of one or more selected audio files.
* Apply ReplayGain (Loudgain) — computes and writes Album + Track ReplayGain across every supported audio file in a folder.
* Report Tag/Filename Mismatches — scans a folder and reports every file whose embedded tags disagree with the filename (a check that the moOde display is correct).
* Write Tags from Folder/File Names — losslessly rewrites Artist/Album/Year/Title/TrackNumber tags from the naming convention, fixing crosswired or missing tags.

Note: These actions print their results to the terminal (or a popup) rather than writing log files. That is a deliberate design choice — right-click actions are meant to be quick spot checks, and they leave no stray files behind in your music folders.

-- Important: replace YOURUSERNAME

Every `.nemo_action` file in this guide contains an `Exec=` line with a `<YOURUSERNAME>` placeholder, for example `/home/<YOURUSERNAME>/.local/bin/verify-album-sha512`. Before the actions will run, replace `<YOURUSERNAME>` with your actual Linux username in **each** action file. Do not paste the placeholder literally — Nemo will simply do nothing if the path does not exist.

This is the single most common reasons the actions appear not to work for someone new. If a right-click action silently fails, check that you replaced the placeholder and that the script exists at the path you gave it.

---

02. Requirements

---

* Linux Mint with the Nemo file manager.
* Terminal access.
* sha512sum (included by default on most distros).
* ffprobe (for the Show ReplayGain action; ships with ffmpeg).
* zenity (for the Show ReplayGain popup; usually preinstalled on Linux Mint).
* loudgain (for the Apply ReplayGain action). Install with:

--- Bash Script Start ---
```bash

sudo apt install loudgain

```
--- Bash Script End ---

-- Expected folder structure

The SHA512 actions rely on the two-level manifest convention used across the suite:

```
Artist Folder/
├── ARTIST.sha512sums.txt
└── Album Folder/
    └── ALBUM.sha512sums.txt
```

The ReplayGain actions work on any folder containing supported audio files (FLAC, MP3, M4A, OGG, Opus, and others).

---

03. How the Actions Fit the Suite

---

The moOde Library Integrity Suite uses two complementary layers of protection:

* **Artist level** — the quick check. `ARTIST.sha512sums.txt` stores one hash per album directory, so verifying an entire artist is a single fast operation.
* **Album level** — the thorough check. `ALBUM.sha512sums.txt` stores one hash per audio file, so it verifies the audio tracks themselves in detail.

The Nemo actions put both layers on the right-click menu, alongside the ReplayGain tools, so you never need to open a terminal for routine checks.

---

04. Part 1 — Verify ALBUM SHA512 Checksums

---

The album action verifies each track file listed in `ALBUM.sha512sums.txt` inside a single album folder.

-- Step 1 — Create the album verification script

--- Bash Script Start ---
```bash

nano ~/.local/bin/verify-album-sha512

```
--- Bash Script End ---

-- Step 2 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash

echo "==================================================="
echo " Verifying ALBUM SHA512 Checksums"
echo "==================================================="
echo

# If a file was dropped onto the script, change to its directory.
if [ -n "$1" ]; then
    cd "$(dirname "$1")" || exit 1
fi

if [ -f "ALBUM.sha512sums.txt" ]; then
    stdbuf -oL sha512sum -c ALBUM.sha512sums.txt 2>&1 |
    awk -F': ' '
        NF == 2 {
            printf "%-8s %s\n", $2, $1
            next
        }
        { print }
    '
else
    echo "MISSING  ALBUM.sha512sums.txt"
fi

echo
echo "Verification Complete."
echo
read -rp "Press Enter to close..."

```
--- nano Paste Script End ---

-- Step 3 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 4 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/verify-album-sha512
ls -l ~/.local/bin/verify-album-sha512   # expect permissions starting with -rwx

```
--- Bash Script End ---

-- Step 5 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/verify-album-sha512.nemo_action

```
--- Bash Script End ---

-- Step 6 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Name=Verify ALBUM SHA512 Checksums
Comment=Check track checksums in ALBUM.sha512sums.txt
Exec=/home/<YOURUSERNAME>/.local/bin/verify-album-sha512 %F
Selection=s
Extensions=txt;
Conditions=exact-name ALBUM.sha512sums.txt;
Icon-Name=dialog-information
Terminal=true
Active=true

```
--- nano Paste Script End ---

-- Step 7 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

05. Part 2 — Verify ARTIST SHA512 Checksums

---

The artist action verifies each album directory listed in `ARTIST.sha512sums.txt` inside an artist folder.

-- Step 8 — Create the artist verification script

--- Bash Script Start ---
```bash

nano ~/.local/bin/verify-artist-sha512

```
--- Bash Script End ---

-- Step 9 — Clear old contents (if replacing an existing script)

In nano, hold `Ctrl+K` until the file is empty.

-- Step 10 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash

if [ -n "$1" ]; then
    cd "$(dirname "$1")" || exit 1
fi

echo "==================================================="
echo " Verifying ARTIST SHA512 Checksums"
echo "==================================================="
echo

if [ -f "ARTIST.sha512sums.txt" ]; then
    artist=$(basename "$PWD")
    echo "=== $artist ==="
    echo

    while IFS= read -r line || [ -n "$line" ]; do
        [ -z "$line" ] && continue

        # Extract hash (first word) and album directory (rest of the line)
        stored_hash=$(awk '{print $1}' <<< "$line")
        album=$(sed 's/^[^ ]*[ ]*//' <<< "$line")

        if [ ! -d "$album" ]; then
            printf "%-10s %s\n" "MISSING" "$album"
            continue
        fi

        actual_hash=$(
            cd "$album" &&
            find . -type f ! -name "ALBUM.sha512sums.txt" -print0 |
            LC_ALL=C sort -z |
            xargs -0 sha512sum |
            sha512sum |
            cut -d' ' -f1
        )

        if [ "$stored_hash" = "$actual_hash" ]; then
            printf "%-10s %s\n" "OK" "$album"
        else
            printf "%-10s %s\n" "MISMATCH" "$album"
        fi
    done < ARTIST.sha512sums.txt
else
    printf "%-10s %s\n" "MISSING" "ARTIST.sha512sums.txt"
fi

echo
echo "Verification Complete."
echo
read -rp "Press Enter to close..."

```
--- nano Paste Script End ---

-- Step 11 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 12 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/verify-artist-sha512
ls -l ~/.local/bin/verify-artist-sha512   # expect permissions starting with -rwx

```
--- Bash Script End ---

-- Step 13 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/verify-artist-sha512.nemo_action

```
--- Bash Script End ---

-- Step 14 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Name=Verify ARTIST SHA512 Checksums
Comment=Check album directory checksums in ARTIST.sha512sums.txt
Exec=/home/<YOURUSERNAME>/.local/bin/verify-artist-sha512 %F
Selection=s
Extensions=txt;
Conditions=exact-name ARTIST.sha512sums.txt;
Icon-Name=dialog-information
Terminal=true
Active=true

```
--- nano Paste Script End ---

-- Step 15 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

06. Part 3 — Show ReplayGain

---

The Show ReplayGain action reads the current ReplayGain tags of one or more selected audio files and displays them. It is read-only and never modifies anything.

It uses ffprobe to read tags uniformly across FLAC (Vorbis comments), MP3 (ID3v2 TXXX frames), and M4A (MP4 freeform atoms). These formats store `REPLAYGAIN_TRACK_GAIN` / `_PEAK` and `REPLAYGAIN_ALBUM_GAIN` / `_PEAK` under the same key names when written by loudgain, so one code path covers all.

-- Step 16 — Create the script

--- Bash Script Start ---
```bash

nano ~/.local/bin/show-replaygain.sh

```
--- Bash Script End ---

-- Step 17 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash
# Show ReplayGain tags for one or more selected FLAC/MP3/M4A files.
# Called from a Nemo Action (see show-replaygain.nemo_action).
#
# Uses ffprobe to read tags uniformly across formats:
#   FLAC -> Vorbis comments, MP3 -> ID3v2 TXXX frames, M4A -> MP4 freeform atoms
# All three store REPLAYGAIN_TRACK_GAIN / _PEAK and REPLAYGAIN_ALBUM_GAIN / _PEAK
# under the same key names when written by loudgain, so one code path covers all.

get_tag() {  # $1 = ffprobe tag dump, $2 = tag name
    printf '%s\n' "$1" | grep -i "^TAG:${2}=" | head -n1 | cut -d= -f2-
}

declare -a ROWS=()
ALBUM_GAIN_SEEN=""
ALBUM_PEAK_SEEN=""
ALBUM_ARTIST_SEEN=""
ALBUM_SEEN=""
MISMATCH=0
ARTIST_MISMATCH=0
ALBUM_MISMATCH=0
LAST_TG=""
LAST_TP=""
LAST_AA=""
LAST_AL=""
LAST_NAME=""

for f in "$@"; do
    [ -f "$f" ] || continue
    tags=$(ffprobe -v error -show_entries format_tags -of default=noprint_wrappers=1 "$f" 2>/dev/null)

    tg=$(get_tag "$tags" REPLAYGAIN_TRACK_GAIN)
    tp=$(get_tag "$tags" REPLAYGAIN_TRACK_PEAK)
    ag=$(get_tag "$tags" REPLAYGAIN_ALBUM_GAIN)
    ap=$(get_tag "$tags" REPLAYGAIN_ALBUM_PEAK)
    aa=$(get_tag "$tags" album_artist)
    al=$(get_tag "$tags" album)

    name=$(basename -- "$f")
    ROWS+=("$name" "${tg:-Not set}" "${tp:-Not set}")
    LAST_TG="$tg"; LAST_TP="$tp"; LAST_AA="$aa"; LAST_AL="$al"; LAST_NAME="$name"

    if [ -n "$ag" ]; then
        if [ -z "$ALBUM_GAIN_SEEN" ]; then
            ALBUM_GAIN_SEEN="$ag"; ALBUM_PEAK_SEEN="$ap"
        elif [ "$ag" != "$ALBUM_GAIN_SEEN" ] || [ "$ap" != "$ALBUM_PEAK_SEEN" ]; then
            MISMATCH=1
        fi
    fi

    if [ -n "$aa" ]; then
        if [ -z "$ALBUM_ARTIST_SEEN" ]; then
            ALBUM_ARTIST_SEEN="$aa"
        elif [ "$aa" != "$ALBUM_ARTIST_SEEN" ]; then
            ARTIST_MISMATCH=1
        fi
    fi

    if [ -n "$al" ]; then
        if [ -z "$ALBUM_SEEN" ]; then
            ALBUM_SEEN="$al"
        elif [ "$al" != "$ALBUM_SEEN" ]; then
            ALBUM_MISMATCH=1
        fi
    fi
done

if [ "$#" -eq 1 ]; then
    zenity --info \
        --title="${LAST_AA:-Unknown Artist} — ${LAST_AL:-Unknown Album}" \
        --width=420 \
        --text="File: ${LAST_NAME}
Track Gain: ${LAST_TG:-Not set}
Track Peak: ${LAST_TP:-Not set}
Album Gain: ${ALBUM_GAIN_SEEN:-Not set}
Album Peak: ${ALBUM_PEAK_SEEN:-Not set}"
else
    TEXT="Album Gain: ${ALBUM_GAIN_SEEN:-Not set}    Album Peak: ${ALBUM_PEAK_SEEN:-Not set}"
    if [ "$MISMATCH" -eq 1 ]; then
        TEXT="${TEXT}
⚠ Album Gain/Peak values are NOT consistent across the selected files"
    fi
    if [ "$ARTIST_MISMATCH" -eq 1 ] || [ "$ALBUM_MISMATCH" -eq 1 ]; then
        TEXT="${TEXT}
⚠ Tracks appear to be from multiple different albums or artists"
    fi
    zenity --list \
        --title="${ALBUM_ARTIST_SEEN:-Unknown Artist} — ${ALBUM_SEEN:-Unknown Album}" \
        --width=760 --height=480 \
        --text="$TEXT" \
        --column="Track" --column="Track Gain" --column="Track Peak" \
        "${ROWS[@]}"
fi

```
--- nano Paste Script End ---

-- Step 18 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 19 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/show-replaygain.sh

```
--- Bash Script End ---

-- Step 20 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/show-replaygain.nemo_action

```
--- Bash Script End ---

-- Step 21 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Active=true
Name=Show ReplayGain
Comment=Display ReplayGain tags for selected audio files
Exec=/home/<YOURUSERNAME>/.local/bin/show-replaygain.sh %F
Icon=audio-x-generic
Selection=notnone
Extensions=flac;mp3;m4a;
Quote=double
Dependencies=ffprobe;zenity;

```
--- nano Paste Script End ---

-- Step 22 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

07. Part 4 — Apply ReplayGain (Loudgain)

---

The Apply ReplayGain action computes and writes **Album + Track** ReplayGain across every supported audio file in the folder, using loudgain. It is intended to be run by right-clicking inside (or on) the folder you want to process, and it reports progress in a terminal.

It does not verify ReplayGain afterwards — use the Show ReplayGain action for that. To re-certify an album after adding ReplayGain, run the relevant checksum steps from the Recertification guide.

-- Step 23 — Create the script

--- Bash Script Start ---
```bash

nano ~/.local/bin/apply-replaygain-folder

```
--- Bash Script End ---

-- Step 24 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash
# Apply ReplayGain (Album + Track) to every supported audio file in a folder.
# Called from a Nemo Action (see apply-replaygain-folder.nemo_action).

# If invoked with a folder path, move into it. Loudgain processes the
# current directory, so this works whether you right-click a folder or
# right-click inside the folder you are viewing.
if [ -n "$1" ] && [ -d "$1" ]; then
    cd "$1" || exit 1
fi

echo "==================================================="
echo " Apply ReplayGain (Loudgain)"
echo "==================================================="
echo
echo "Folder: $(pwd)"
echo

if ! command -v loudgain >/dev/null 2>&1; then
    echo "ERROR: loudgain was not found."
    echo "Install it with: sudo apt install loudgain"
    echo
    read -rp "Press Enter to close..."
    exit 1
fi

shopt -s nullglob nocaseglob
files=( *.flac *.mp3 *.m4a *.ogg *.opus *.mp4 *.aac *.ape *.wv *.mpc *.spx )

if [ ${#files[@]} -eq 0 ]; then
    echo "No supported audio files found in this folder."
    echo
    read -rp "Press Enter to close..."
    exit 0
fi

echo "Processing ${#files[@]} audio file(s)..."
echo

loudgain -a -k -s e -L -- "${files[@]}"

rc=$?

echo
echo "----------------------------------------"
if [ "$rc" -eq 0 ]; then
    echo "SUMMARY: ReplayGain applied to ${#files[@]} file(s) (album + track)."
else
    echo "SUMMARY: Loudgain finished with errors (exit code $rc)."
fi
echo "----------------------------------------"
echo

read -rp "Press Enter to close..."

```
--- nano Paste Script End ---

-- Step 25 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 26 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/apply-replaygain-folder

```
--- Bash Script End ---

-- Step 27 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/apply-replaygain-folder.nemo_action

```
--- Bash Script End ---

-- Step 28 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Name=Apply ReplayGain (Loudgain)
Comment=Compute and write Album + Track ReplayGain for all audio files in the folder
Exec=/home/<YOURUSERNAME>/.local/bin/apply-replaygain-folder %P
Selection=notnone
Extensions=dir;
Icon-Name=audio-x-generic
Terminal=true
Active=true
Dependencies=loudgain;

```
--- nano Paste Script End ---

Note: `%P` passes the folder the action was launched in, so the script processes the entire enclosing folder regardless of the exact item you right-click.

-- Step 29 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

08. Part 5 — Report Tag/Filename Mismatches

---

This action scans a folder (recursively, for the checked scope) and reports every audio file where the embedded metadata differs from the filename — flagging wrong titles, wrong track numbers, or missing tags before moOde ever displays them.

It compares case-insensitively and ignores whitespace, so harmless differences in capitalization are not flagged. Files whose name has no `NN - Title` prefix are reported specially rather than falsely matched.

-- Step 30 — Create the script

--- Bash Script Start ---
```bash

nano ~/.local/bin/report-tag-mismatches

```
--- Bash Script End ---

-- Step 31 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash
# Check for mismatches between embedded tags and filenames.
# Lenient comparison: case-insensitive, whitespace-trimmed.
# Handles files whose name has no "NN - " prefix (reported, not falsely matched).

if [ -n "$1" ] && [ -d "$1" ]; then
    cd "$1" || exit 1
fi

START_DIR=$(pwd)
OUTPUT_FILE="$START_DIR/meta-tag-mismatches.md"
TEMP_RESULTS=$(mktemp)
trap 'rm -f "$TEMP_RESULTS"' EXIT

norm() { tr '[:upper:]' '[:lower:]' | tr -s '[:space:]' ' '; }

find . -type f \( -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" \) | sort | while read -r filepath; do
    filename=$(basename "$filepath")
    name_no_ext="${filename%.*}"

    if [[ "$name_no_ext" =~ ^[0-9]+[[:space:]]+-?[[:space:]]+(.+)$ ]]; then
        file_track_num=${BASH_REMATCH[0]%%[^0-9]*}
        file_track_name="${BASH_REMATCH[1]}"
    else
        file_track_num="NONE"
        file_track_name="NONE"
    fi

    meta_json=$(ffprobe -v error -print_format json -show_format "$filepath" 2>/dev/null)
    if [ -n "$meta_json" ]; then
        meta_track_num=$(echo "$meta_json" | jq -r '.format.tags | to_entries[] | select(.key | ascii_downcase == "track" or ascii_downcase == "tracknumber") | .value' | head -1)
        meta_track_name=$(echo "$meta_json" | jq -r '.format.tags | to_entries[] | select(.key | ascii_downcase == "title") | .value' | head -1)
    fi

    if [ -z "$meta_track_num" ]; then meta_track_num="MISSING"; fi
    if [ -z "$meta_track_name" ]; then meta_track_name="MISSING"; fi

    file_num_clean="$file_track_num"
    if [ "$file_track_num" != "NONE" ]; then
        file_num_clean=$(printf '%s' "$file_track_num" | sed -E 's/^0+//')
        [ -z "$file_num_clean" ] && file_num_clean="0"
    fi

    meta_num_clean=$(printf '%s' "$meta_track_num" | sed -E 's/^0*([0-9]+).*/\1/')
    [ -z "$meta_num_clean" ] && meta_num_clean="0"

    file_name_norm=$(printf '%s' "$file_track_name" | norm | xargs)
    meta_name_norm=$(printf '%s' "$meta_track_name" | norm | xargs)

    mismatch=0
    if [ "$file_track_num" = "NONE" ]; then
        mismatch=1
    elif [ "$file_num_clean" != "$meta_num_clean" ]; then
        mismatch=1
    fi

    if [ "$file_track_name" = "NONE" ] || [ "$file_name_norm" != "$meta_name_norm" ]; then
        mismatch=1
    fi

    if [ "$mismatch" -eq 1 ]; then
        artist=$(awk -F/ '{print $(NF-2)}' <<< "$filepath")
        album=$(awk -F/ '{print $(NF-1)}' <<< "$filepath")
        echo "MISMATCH|$artist|$album|$file_num_clean|$file_track_name|$meta_num_clean|$meta_track_name" >> "$TEMP_RESULTS"
    fi
done

{
    echo "# Meta Tag vs Filename Mismatches"
    echo ""
    echo "Generated: $(date)"
    echo "Scope: $START_DIR"
    echo ""
    if [ -s "$TEMP_RESULTS" ]; then
        prev_artist=""; prev_album=""
        sort -t'|' -k2,2 -k3,3 "$TEMP_RESULTS" | while IFS='|' read -r _ artist album file_track file_name meta_track meta_name; do
            if [ "$artist|$album" != "$prev_artist|$prev_album" ]; then
                [ -n "$prev_album" ] && echo ""
                echo "## $artist - $album"
                echo ""
                printf '%s\n' "| File# | File Title | Tag# | Tag Title |"
                printf '%s\n' "|------:|------------|-----:|-----------|"
            fi
            [ "$file_track" = "NONE" ] && file_track="-"
            [ "$meta_track" = "MISSING" ] && meta_track="?"
            printf '| %s | %s | %s | %s |\n' "$file_track" "$file_name" "$meta_track" "$meta_name"
            prev_artist="$artist"; prev_album="$album"
        done
    else
        echo "No mismatches found."
    fi
} > "$OUTPUT_FILE"

echo "Report written to: $OUTPUT_FILE"

```
--- nano Paste Script End ---

-- Step 32 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 33 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/report-tag-mismatches

```
--- Bash Script End ---

-- Step 34 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/report-tag-mismatches.nemo_action

```
--- Bash Script End ---

-- Step 35 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Name=Report Tag/Filename Mismatches
Comment=Scan for files where embedded tags differ from the filename
Exec=/home/<YOURUSERNAME>/.local/bin/report-tag-mismatches %P
Selection=notnone
Extensions=dir;mp3;m4a;flac;
Icon-Name=dialog-information
Terminal=true
Active=true
Dependencies=ffprobe;jq;

```
--- nano Paste Script End ---

Note: `%P` passes the folder the action was launched in, so the report covers the entire enclosing scope.

-- Step 36 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

09. Part 6 — Write Tags from Folder/File Names

---

This is the companion to Part 5: it fixes the mismatches automatically by deriving metadata from the naming convention and writing it losslessly into each file.

Run it from inside an album folder named `YYYY Album Name`, with tracks named `NN - Title.ext`. It infers:

* Artist — from the parent folder name.
* Album year — leading 4-digit year in the album folder name.
* Album name — the folder name with the year stripped off.
* Track number / title — from each filename.

It writes those tags losslessly per format: metaflac (FLAC), eyeD3 (MP3), and AtomicParsley (M4A). M4A is written in place with no re-encoding, so your audio is never altered or recompressed. Files that do not match the `NN - Title` pattern are skipped and reported, never guessed.

-- Naming convention is REQUIRED

This tool only works because the folder and filenames themselves already carry the correct metadata (year in the folder name, track number and title in the filename). That is a deliberate safety property of the library: even if every tag is destroyed, the full metadata can be rebuilt from the file tree alone.

Therefore the convention is a **required precondition**, not a suggestion. Do **not** run this on an album folder that does not follow it:

* The album folder name **must** begin with a 4-digit year, followed by a space: `2020 Demo Album`. Otherwise there is nowhere for the year to come from.
* Track files **must** be named `NN - Title.ext` (a track number, a space, a dash, a space, then the title).

The script refuses to guess. If the album folder has no leading year, it aborts and tells you, rather than write an album with a missing or wrong year that would silently break the rebuild-from-tags property. If a track name does not match the `NN - Title` pattern, that file is skipped and reported so you can fix its name first.

-- Step 37 — Create the script

--- Bash Script Start ---
```bash

nano ~/.local/bin/write-tags-from-names

```
--- Bash Script End ---

-- Step 38 — Paste the script

--- nano Paste Script Start ---
```bash

#!/bin/bash
# Write tags (Artist / Album / Year / Title / TrackNumber) from folder + filename.
# Run from inside an album folder. Names expected:
#   Album folder : "YYYY Album Name"
#   Audio files  : "NN - Title.ext"
# Files that don't match the "NN - Title" pattern are skipped and reported.
# All writes are LOSSLESS (metaflac / eyeD3 / AtomicParsley) - no re-encoding.
set -o pipefail

START_DIR=$(pwd)

artist=$(awk -F/ '{print $(NF-1)}' <<< "$START_DIR")
album_dir=$(basename "$START_DIR")
if [[ "$album_dir" =~ ^([0-9]{4})[[:space:]]+(.+)$ ]]; then
    album_year="${BASH_REMATCH[1]}"
    album_name="${BASH_REMATCH[2]}"
else
    echo "==================================================="
    echo " Write Tags from Folder / File Names"
    echo "==================================================="
    echo
    echo "ABORT: The album folder name must begin with a 4-digit"
    printf 'year followed by a space, e.g. "2020 Demo Album".\n'
    printf 'Got: %s\n' "$album_dir"
    echo
    echo "The year has to come from the folder name. Rename the"
    echo "folder to 'YYYY Album Name', then run this again."
    echo
    read -rp "Press Enter to close..."
    exit 1
fi

echo "==================================================="
echo " Write Tags from Folder / File Names"
echo "==================================================="
printf 'Artist : %s\n' "$artist"
printf 'Album  : %s\n' "$album_name"
printf 'Year   : %s\n' "$album_year"
echo

SKIPPED=()
WRITTEN=0
FAILED=0

while IFS= read -r -d '' filepath; do
    filename=$(basename "$filepath")
    ext="${filepath##*.}"
    name_no_ext="${filename%.*}"

    if [[ "$name_no_ext" =~ ^([0-9]+)[[:space:]]+-?[[:space:]]+(.+)$ ]]; then
        track_num_str="${BASH_REMATCH[1]}"
        track_num=$(( 10#${track_num_str} ))
        track_name="${BASH_REMATCH[2]}"
    else
        SKIPPED+=("$filepath")
        printf 'SKIP    %-14.14s %s (name has no "NN - Title")\n' "$ext" "$filename"
        continue
    fi

    rc=0
    case "${ext,,}" in
        flac)
            metaflac \
                --remove-tag=ARTIST \
                --remove-tag=ALBUMARTIST \
                --remove-tag=ALBUM \
                --remove-tag=DATE \
                --remove-tag=YEAR \
                --remove-tag=TITLE \
                --remove-tag=TRACKNUMBER \
                "$filepath" 2>/dev/null
            metaflac \
                --set-tag="ARTIST=$artist" \
                --set-tag="ALBUMARTIST=$artist" \
                --set-tag="ALBUM=$album_name" \
                --set-tag="DATE=$album_year" \
                --set-tag="YEAR=$album_year" \
                --set-tag="TITLE=$track_name" \
                --set-tag="TRACKNUMBER=$track_num" \
                "$filepath"
            rc=$?
            ;;
        mp3)
            eyeD3 --artist "$artist" \
                --album-artist "$artist" \
                --album "$album_name" \
                --release-year "$album_year" \
                --title "$track_name" \
                --track "$track_num" \
                "$filepath" >/dev/null 2>&1
            rc=$?
            ;;
        m4a)
            args=()
            [ -n "$album_year" ] && args+=(--year "$album_year")
            AtomicParsley "$filepath" \
                --artist "$artist" \
                --albumArtist "$artist" \
                --album "$album_name" \
                --title "$track_name" \
                --tracknum "$track_num" \
                "${args[@]}" \
                --overWrite >/dev/null
            rc=$?
            ;;
        *)
            SKIPPED+=("$filepath")
            printf 'SKIP    %s (unsupported extension)\n' "$filename"
            continue
            ;;
    esac

    if [ "$rc" -eq 0 ]; then
        printf 'OK      %-4s %02d - %s\n' "$ext" "$track_num" "$track_name"
        ((WRITTEN++))
    else
        printf 'FAILED  %-4s %s\n' "$ext" "$filename"
        ((FAILED++))
    fi
done < <(find . -maxdepth 1 -type f \( -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" \) -print0 | sort -z)

echo
echo "----------------------------------------"
printf 'SUMMARY: %d written, %d failed, %d skipped.\n' "$WRITTEN" "$FAILED" "${#SKIPPED[@]}"
echo "----------------------------------------"
echo
printf 'Reminder: Tags were changed, so ALBUM.sha512sums.txt / ARTIST.sha512sums.txt\n'
printf 'are now stale. Re-run "Regenerate ALBUM/ARTIST Checksum" after tagging.\n'
echo
read -rp "Press Enter to close..."

```
--- nano Paste Script End ---

-- Step 39 — Save the script

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

-- Step 40 — Make it executable

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/write-tags-from-names

```
--- Bash Script End ---

-- Step 41 — Create the action file

--- Bash Script Start ---
```bash

nano ~/.local/share/nemo/actions/write-tags-from-names.nemo_action

```
--- Bash Script End ---

-- Step 42 — Paste the action

--- nano Paste Script Start ---
```ini

[Nemo Action]
Name=Write Tags from Folder/File Names
Comment=Losslessly write Artist/Album/Year/Title/Track tags from the naming convention
Exec=/home/<YOURUSERNAME>/.local/bin/write-tags-from-names %P
Selection=notnone
Extensions=dir;
Icon-Name=accessories-text-editor
Terminal=true
Active=true
Dependencies=AtomicParsley;eyeD3;metaflac;

```
--- nano Paste Script End ---

-- Step 43 — Save

In nano: `Ctrl+O`, `Enter`, `Ctrl+X`

---

10. Part 7 — Restart Nemo

---

--- Bash Script Start ---
```bash

nemo -q

```
--- Bash Script End ---

---

11. Part 8 — Testing

---

-- Test album SHA512:

     1. Open an album folder containing: ALBUM.sha512sums.txt
     2. Right-click it → Verify ALBUM SHA512 Checksums
     3. Expect `OK` output for each file

-- Test artist SHA512:

     1. Open an artist folder containing: ARTIST.sha512sums.txt
     2. Right-click it → Verify ARTIST SHA512 Checksums
     3. Expect `OK` output for each album

Known-working test:

Folder: /media/<username>/<drive>/ArtistName
File:   ARTIST.sha512sums.txt

Result:

=== ArtistName ===
OK  AlbumName

-- Test Show ReplayGain:

     1. Select one or more FLAC/MP3/M4A files
     2. Right-click → Show ReplayGain
     3. Expect a popup listing track/album gain and peak values

-- Test Apply ReplayGain:

     1. Open a folder with supported audio that lacks ReplayGain tags
     2. Right-click inside the folder → Apply ReplayGain (Loudgain)
     3. Expect a terminal showing loudgain progress and a `SUMMARY:` line
     4. Right-click the files → Show ReplayGain to confirm tags were written

---

12. Checksum File Formats

---

`ARTIST.sha512sums.txt` — one line per album: `SHA512_HASH  Album Directory Name`

Example:

b47535abe91048fd9f224d1d6ad74c3056b006491f74de9c0227b646b1a84e861422a4bcd85a7c33854ecf619074cd919298cb5768a0b933a578a207553631b8  AlbumName

The album name is a directory, not a file — the artist script computes a hash-of-hashes across the album's own contents (excluding `ALBUM.sha512sums.txt` itself) and compares it against the stored value.

`ALBUM.sha512sums.txt` — one line per track: `SHA512_HASH  filename.ext`, generated by sha512sum.

---

13. File Locations

---

| Item         | Path                          |
|--------------|-------------------------------|
| Scripts      | ~/.local/bin/                 |
| Nemo actions | ~/.local/share/nemo/actions/  |

Working `Exec=` format (do not change without testing): `Exec=/home/<YOURUSERNAME>/.local/bin/script-name %F` (or `%P` for the folder action).

The SHA512 scripts require `$1` because Nemo launches actions from an unknown working directory; the script `cd`s into the directory containing the selected manifest.

---

14. Troubleshooting

---

1. Script works from terminal but the Nemo right-click action does nothing.

* Confirm the action file is in: ~/.local/share/nemo/actions/
* Confirm the `Exec=` line matches your real username.
* Restart Nemo: nemo -q

2. "MISSING ALBUM.sha512sums.txt" or "MISSING ARTIST.sha512sums.txt".

The manifest has not been created in that folder. Run the checksum generation step from the Recertification or SHA512 Library guide first.

3. An album/artist shows MISMATCH.

The audio has changed (or bit rot / an incomplete copy). Do not regenerate the hash to "fix" it — restore the file from a known-good backup, then re-certify.

4. ReplayGain action not applying.

Confirm loudgain is installed and the `Dependencies=loudgain;` line is present. The Apply ReplayGain action only works inside a folder with supported audio files.

---

15. Backup

---

Keep copies of:

* ~/.local/bin/verify-album-sha512
* ~/.local/bin/verify-artist-sha512
* ~/.local/bin/show-replaygain.sh
* ~/.local/bin/apply-replaygain-folder
* ~/.local/bin/report-tag-mismatches
* ~/.local/bin/write-tags-from-names
* ~/.local/share/nemo/actions/verify-album-sha512.nemo_action
* ~/.local/share/nemo/actions/verify-artist-sha512.nemo_action
* ~/.local/share/nemo/actions/show-replaygain.nemo_action
* ~/.local/share/nemo/actions/apply-replaygain-folder.nemo_action
* ~/.local/share/nemo/actions/report-tag-mismatches.nemo_action
* ~/.local/share/nemo/actions/write-tags-from-names.nemo_action

---

16. Restore from Backup

---

Use after a Linux reinstall, system rebuild, or move to another machine.

    1. Copy the scripts to ~/.local/bin/
    2. Copy the .nemo_action files to ~/.local/share/nemo/actions/
    3. Make scripts executable:

--- Bash Script Start ---
```bash

chmod +x ~/.local/bin/verify-album-sha512
chmod +x ~/.local/bin/verify-artist-sha512
chmod +x ~/.local/bin/show-replaygain.sh
chmod +x ~/.local/bin/apply-replaygain-folder
chmod +x ~/.local/bin/report-tag-mismatches
chmod +x ~/.local/bin/write-tags-from-names

```
--- Bash Script End ---

    4. Restart Nemo:

--- Bash Script Start ---
```bash

nemo -q

```
--- Bash Script End ---

    5. Test all seven actions as described in Section 11 — Part 8: Testing.

Setup is confirmed restored once the actions appear in the Nemo right-click menu.

\-------------------------------------------------------------------

-- Disclaimer

This guide was developed through iterative collaborative effort between ChatGPT, Claude, Gemini, Mistral and the user. I cannot thank OpenCode project enough. I was about to give up on the other four (well, actually I did) when I came across OpenCode. I run a 10+ year old laptop yet OpenCode ran perfectly well, offloading the heaving lifting to an offsite server.

https://opencode.ai/
