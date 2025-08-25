# README

### 1) Here-string: `<<<`

“Take this variable’s contents and pretend it’s a tiny file on stdin.”

* In your line, it’s the bit `<<< "$image"`.
* Effect: `sed` reads the **value of `$image`** as its input (no pipe, no echo).
* Why it exists: zero-overhead way to feed a **single string/variable** to a command.

### 2) Process substitution: `<( … )`

“Run this command in the background and give me a **file-like handle** to its output.”

* In your line, it’s `< <( sed … )`.
* `<( … )` expands to something like `/dev/fd/63`.
* You then use a normal input redirection `<` so that `read` **reads from that pseudo-file**.

> Important nuance: `read -r name tag < <( … )` needs **both**:
>
> * `<( … )` produces the file descriptor, and
> * the leading `<` tells `read` to **read from it**.
>   Without the extra `<`, `read` would just see an extra argument, not input.

### 3) Plain input redirection: `<`

“Read stdin from this file (or FD).”

* You combine it with process substitution: `read … < <(cmd)`.

---

## The split + parse line (the core of it)

```bash
read -r name tag < <( sed -r 's|^(.*):(.*)|\1 \2|' <<< "$image" )
```

Breakdown:

* `<<< "$image"`
  Feeds the **string** (e.g., `apache:v1.0.0`) to `sed` as stdin.

* `sed -r 's|^(.*):(.*)|\1 \2|'`

  * `-r`: use extended regex.
  * Pattern `^(.*):(.*)`:

    * `^` anchor at start.
    * First `(.*)` = everything up to the **last** colon (because `.*` is greedy).
    * `:` literal colon delimiter.
    * Second `(.*)` = everything after that colon (the tag).
  * Replacement `\1 \2` inserts a **space** between the two captured groups.
  * Output becomes: `name[space]tag` (e.g., `apache v1.0.0`).

  Why “last colon”? Greedy `.*` grabs as much as possible.
  That’s good here: it handles `registry:5000/repo:v1.0.0` by splitting at the **tag colon**, not the port colon.

* `<( … )`
  Wraps `sed`’s stdout as a **file-like** path (e.g., `/dev/fd/63`).

* `read -r name tag < …`

  * `read`: splits the incoming line on IFS (default: spaces/tabs/newlines).
  * First word → `name`, second word → `tag`.
  * `-r`: don’t treat backslashes as escapes (safer, predictable).

Net effect: for each `$image`, you end with:

* `name="apache"`
* `tag="v1.0.0"`

---

## Array handling lines

```bash
tags+=("${tag}")
names+=("${name}")
```

* Appends each parsed piece into parallel arrays.
* Same index `i` in `names[i]` and `tags[i]` refers to parts from the **same** original image.

---

## Iterating by **indices** (why `${!array[@]}`)

```bash
for i in "${!images[@]}"; do
    echo "${names[i]}:${tags[i]}"
done
```

* `${!images[@]}` = **all indices** currently used in `images`.
* Printing by index keeps `names[i]` and `tags[i]` aligned 1:1.
* `${array[@]}` would give values; `${!array[@]}` gives **indices**.

---

## Analogy (because it helps wire it in your head)

* Think of `<<< "$image"` as handing a **note card** with one line to `sed`.
* `sed` rewrites the note so the two fields are separated by **one space**.
* `<( … )` turns `sed`’s output into a **tap** labeled `/dev/fd/63`.
* `read … < /dev/fd/63` is you plugging a **hose** from that tap straight into `read`.
  `read` then pours the first word into **bucket `name`** and the second into **bucket `tag`**.

---

## Why your comments are correct

* “`<<<` Redirecting VARIABLES INTO SED” — yes: here-strings feed the variable’s text to `sed` as stdin.
* “`< <( command )` lets you have the output of the command as input and then you can use another redirect (`<`) to feed it into `read`” — exactly.
  `<( … )` builds the pseudo-file; the leading `<` tells `read` to read **from** it.

That’s the whole dance: **variable → here-string → sed → process substitution → read** → arrays → print. Clean.
