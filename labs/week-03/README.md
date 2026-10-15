# Week 3 record

Your name: 20099070
Date: 08/10/2026

Fill this in as you work, rather than at the end. Where you are unsure, write
that you are unsure and say why. That is worth more than a confident sentence you
cannot support.

---

## 1. The finding I chose

State which of the three the tool reported.

- File: `src/handlers/directory.js`
- Line: lines 8 to 11 (the `const sql = ...` that builds the directory search)
- What the tool said about it: Rule `sql-built-by-joining-text`, severity ERROR,
  confidence MEDIUM. It said the SQL statement is built by joining pieces of
  text together, and that if any of those pieces came from outside the
  application, the database would read it as part of the command rather than as
  a value.

## 2. What an assistant told me

Say which assistant you asked, and what it said in a sentence or two.

- Assistant used: Claude (Anthropic)
- Its explanation, in your own words: The search word is joined into the SQL
  text with `+` instead of being passed as a separate value, so the database
  cannot tell the typed word apart from the command around it. It said this is
  the shape of a classic SQL injection (CWE-89), but told me not to assume it
  was real until I checked where `q` comes from and whether anything cleans it.
- One thing it asserted that I had not verified at that point: that the value
  reaches the query with nothing checking or escaping it. The tool and the
  assistant both only read the code; neither had run Atrium.

## 3. What the code shows

Answer all four. If you cannot answer one, say so.

**Where does the data come from?** From outside the application. In
`showDirectory` (line 19) the value is `req.query.q`, the `q` from the address
bar, e.g. `/directory?q=Nolan`. It is passed into `searchDirectory(db, q)`.

**What happens to it on the way?** Almost nothing. Line 19 only checks it is a
string (otherwise it becomes `""`). The value is not trimmed, escaped or
checked against a list. The same text that arrives is the text that is used.

**Where does it become dangerous?** At lines 9 to 11, where `q` is joined into
the SQL string with `+`, and then at line 13, `db.prepare(sql)`, where that
finished string is handed to the database as the command. The neighbouring
`listColleagues` (line 4) is safe because it joins no outside value.

**What stands in the way?** Nothing that I can find on this path. There is no
parameter placeholder (`?`), no escaping and no allow-list between `req.query.q`
and `db.prepare`. The other two findings do use a `?` placeholder, so they keep
the value apart from the command; this one does not.

## 4. What the running application shows

Record both. A single result on its own proves nothing.

**Ordinary case**

- What I entered: `Nolan` in the directory search box.
- What came back: one row, Alice Nolan. The query ran as
  `... LIKE '%Nolan%' OR department LIKE '%Nolan%'`.

**The case I was testing for**

- What I entered: the lab's example, `' OR '1'='1`
- What came back: all three colleagues (Alice Nolan, Bob Keane, Morgan Doyle),
  not just the one whose name matched. The typed quote ended the text value
  early, so the rest was read as part of the command and the WHERE condition
  became always-true.
- How this differs from the ordinary case: the code did not change between the
  two searches. Only the typed text changed, and it changed how many rows the
  database returned. In the ordinary case the typed text stayed a value; in this
  case it became part of the command.

## 5. My answer

Delete the two that do not apply.

**Real**

**Why, in two or three sentences.** The directory search in this version of
Atrium puts the typed word straight into a database command without keeping it
apart from the command. A word with a quote placed correctly changes what the
command means: searching for the lab's example returned every colleague instead
of the one that matched. I traced it in the code, saw no control on the path,
and reproduced the change on the running lab, so someone else could follow the
same steps and see the same result.

**What would change my mind.** If I found a check I had missed between
`req.query.q` and `db.prepare` (for example a placeholder or escaping added
somewhere I did not look), or if the directory page turned out to be
unreachable, this would move back down to "not real" or "not settled".

**How far this answer reaches.** I have shown this only for the directory search
page, on localhost, with the lab's seed data, in this version of Atrium. It says
nothing about the other two findings, nothing about the rest of the application,
and nothing yet about how serious it is in a real deployment. It is one page,
one finding.

## 6. Back to the assistant

The thing you noted in section 2, that you had not verified at the time.

- Did I check it? Yes. I traced `q` from `req.query.q` to `db.prepare` and ran
  both searches on the lab.
- Was it right? Yes. There is no control on that path, and the typed text
  changed the command when I ran it. The assistant was right, but I only know
  that because I checked it rather than because it said so.

---

## Optional, if you had time

The other two findings matched the same rule. Why are they not the same situation?
Two sentences.

The matches in `reset.js` and `resources.js` join text with `+` too, so the rule
fired, but each puts a `?` placeholder where a value goes, which keeps the value
apart from the command. The directory finding has no placeholder and takes its
value from `req.query.q`, so the rule matched the same shape in all three places
but only this one lets outside text reach the database as part of the command.
