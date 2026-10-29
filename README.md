# Bug Hunt Blog ✍️🐞

A small multi-user blog. People log in, write a draft, **publish** it, and **delete**
posts they no longer want. It works… mostly. Six bugs are hiding in it.

This is the second project in the series. The first one (Bug Hunt Library)
taught you to read tracebacks and use `breakpoint()`. This one goes further:

- bugs where the app **says it worked but didn't**
- **off-by-one** bugs
- **security** bugs, where nothing crashes and everything looks fine
  until the wrong person does something

---

## 1. Run it

**In GitHub Codespaces:** open the repo → **Code → Codespaces → Create codespace**.
Everything installs and the site starts by itself.

**On your own computer:**
```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python manage.py migrate
python manage.py seed              # two users + sample posts
python manage.py runserver
```

**Sample accounts** (created by `seed`):

| Username | Password |
|---|---|
| `alice` | `debugme123` |
| `bob` | `debugme123` |

**Tip for security bugs:** you'll need to be two people at once. Log in as
**alice in a normal window** and as **bob in a private/incognito window**.
Each window keeps its own login.

Want a clean slate after breaking things? Run `python manage.py seed` again.

---

## 2. The method (same six steps as before)

1. **Reproduce:** make it happen on purpose, and write down the steps.
2. **Read:** read the full error, or describe exactly what's wrong if there's no error.
3. **Locate:** find where good data turns into bad data.
4. **Hypothesize:** say specifically what you think is wrong.
5. **Test the idea:** use `print`, the shell, `breakpoint()`, or the admin.
6. **Fix and verify:** change one thing, then run the ticket's test **and** the whole suite.

---

## 3. New tools for this project

### A. Check the database, not the screen
A success message is just text the code chose to show. It proves nothing.
To see what was *actually* saved, use either of these:

**Django admin:**
```bash
python manage.py createsuperuser
```
Then visit `/admin/` → Posts. You'll see the real status of each post.

**Django shell:**
```bash
python manage.py shell
```
```python
>>> from blog.models import Post
>>> Post.objects.values_list("title", "status", "published_at")
```

### B. Read database errors
`IntegrityError` means the database refused to save something because it broke
a rule, like "this column can't be empty" or "this value must be unique".
The message names the table and the column. That's your clue.

### C. Boundary values for off-by-one bugs
When counting or slicing goes wrong, test the edges: the **first** item, the
**last** item, page **1**, the **last** page. Write down what you *expect* on
each page before you look. In the shell:
```python
>>> posts = Post.objects.filter(status="published").order_by("-published_at")
>>> list(posts[0:5])    # what should page 1 be?
```

### D. Think like an attacker
For every action, ask two questions: *who* is allowed to do this, and *where*
does the code check that? Logging in proves who you are. It does **not**
prove you own a post. Try every URL as:
1. the owner (alice)
2. another user (bob, in the incognito window)
3. a stranger (logged out)

Anything bob or a stranger can do that they shouldn't is a bug, even
though nothing crashed.

### E. GET vs POST
- **GET** means "show me something". Browsers, link previews, search crawlers and
  prefetchers send GET requests freely, so GET must **never change data**.
- **POST** means "do something". Use it for anything that creates, changes or
  deletes data, and protect it with `{% csrf_token %}`.

### F. The VS Code debugger (Codespaces)
In **Run and Debug**, pick **"Debug Django (runserver)"**, then click in the margin
next to a line in `blog/views.py` to add a red breakpoint dot. Use the page in
your browser: execution pauses on that line, and you can hover over any
variable to see its value. Use **Step Over** (F10) to run one line at a time.

### G. Tests are the scoreboard
```bash
python manage.py test               # all tickets
python manage.py test blog.tests.Ticket4
```
8 of the 10 tests fail at the start. The other 2 already pass, and they **must
stay passing**: they check that your fix doesn't go too far, such as
stopping alice from seeing her own draft.

---

## 4. The tickets

Work through them **in order**.

> **Ticket 1 — "Saving a new post crashes."**
> *From alice:* "I wrote a whole post, clicked Save draft, and got an error page
> with 'IntegrityError'. My post is gone."
> 🛠 Skill: reading database errors; how `ModelForm.save()` works

> **Ticket 2 — "Publish says it worked, but it didn't."**
> *From alice:* "I clicked Publish. It said '…is live!' in green. But the post
> still says Draft, and it's not on the home page."
> 🛠 Skill: checking the database, not the message (tool 3A)

> **Ticket 3 — "My newest posts never show on the home page."**
> *From alice:* "The home page starts with older stuff. The post I published this
> morning isn't on any page at all. I clicked through every one."
> 🛠 Skill: off-by-one errors and boundary values (tool 3C)

> **Ticket 4 — "Someone read my draft!"**
> *From alice:* "My 'Secret draft' isn't published, but bob sent me a screenshot of it.
> He said he just guessed the URL."
> 🛠 Skill: thinking like an attacker (tool 3D)

> **Ticket 5 — "Bob deleted my post."**
> *From alice:* "One of my posts is gone. Bob says he was 'just testing'."
> 🛠 Skill: authorization (logged in is not the same as allowed)

> **Ticket 6 — "Posts disappear when someone just *opens* a link."**
> *From alice:* "I pasted my delete link into a group chat to ask if the button looked
> OK. The chat app loaded a link preview… and the post was deleted."
> 🛠 Skill: GET vs POST (tool 3E). Look around the templates folder; the previous
> developer started on a fix.

Stuck for 15+ minutes? `HINTS.md` has three hints per ticket.
`SOLUTIONS.md` is for checking your work afterwards.

---

## 5. Keep a bug log

Add a line to `BUGLOG.md` after each fix:

```
Ticket 2 | Symptom: "published!" message but post still draft
         | Cause: view changed the object in memory but never called save()
         | Found by: admin showed status=draft | Fix: post.save()
```

Tickets 4–6 make excellent interview stories: "I found a security hole where…"

---

## 6. Finished? Level up

- **Bonus bug:** visit `/?page=banana`. What happens, and what *should* happen?
  Fix it, and write a test first.
- Rewrite `post_list` with Django's built-in `Paginator`. Do the Ticket 3 tests still pass?
- Add an **Edit post** feature. Before writing it, list who should and shouldn't be
  able to edit, and write those as tests. (You just learned what happens if you don't.)
