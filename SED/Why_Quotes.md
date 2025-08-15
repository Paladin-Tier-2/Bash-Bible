Because the **shell** (bash/zsh) messes with your text *before* `sed` ever sees it.
Quoting tells the shell: **“hands off—pass this to `sed` unchanged.”**

### What the shell would otherwise do (and break)

* **Word splitting:** spaces split your script into multiple args.

  ```bash
  sed s/CustomLog /proc/self/fd/1/ combined   # ← splits at spaces → garbage
  ```
* **Globbing:** `* ? [ ]` expand to filenames.

  ```bash
  sed s/[a-z]*/X/g file    # shell expands [a-z]* to files in cwd 😬
  ```
* **History expansion (bash):** `!g`, `!$`, etc. blow up.

  ```bash
  sed s/.$/!g/ file        # bang gets eaten by history unless quoted
  ```
* **Variable/command substitution:** `$VAR` or `$(...)` get expanded by the shell—great if you *want* that, awful if you don’t.

### Which quotes, when

* **Single quotes `'...'` (strong quoting):** pass **exactly** what you wrote.
  Use this by default for `sed` scripts.

  ```bash
  sed -E 's|^[[:space:]]*User[[:space:]]+.*$|User www-data|' /etc/apache2/apache2.conf
  # shell does not split, glob, or expand anything; sed gets it raw
  ```
* **Double quotes `"..."` (weak quoting):** allow `$VAR`, `$(...)`, backticks to expand; still prevent word splitting/globbing.
  Use only when you *need* expansion.

  ```bash
  sock="/run/mysqld/mysqld.sock"
  sed -ri "s|^[[:space:]]*mysqli\.default_socket[[:space:]]*=.*|mysqli.default_socket = $sock|" /etc/php/8.3/apache2/php.ini
  # $sock expands; regex stays intact
  ```
* **ANSI-C quotes `$'...'`:** let you write literal control chars (`\n`, `\t`) *in the shell*.

  ```bash
  sed -E $'s/\t/    /g' file   # replace TABs with 4 spaces
  ```

### Subtle but important

* Things like `\1` (backref) and `&` (whole match) are **`sed` features**, not shell ones. You quote to stop the **shell** from touching them so `sed` can.
* `#` starts a shell comment **unless quoted**. Many `sed` scripts use `#` as a delimiter—quote them.
* Bracket classes `[[:space:]]` look like globs to the shell—quote them.

### Practical rules

1. **Default:** wrap the `sed` script in **single quotes**.
2. **Need `$VAR`?** switch to **double quotes** (and be mindful of any `$` that should stay literal—escape as `\$` if needed).
3. **Need tabs/newlines literally?** use **`$'...'`**.

**TL;DR:** Quotes aren’t for `sed`; they’re armor against the shell so your regex/replacement arrives intact.
