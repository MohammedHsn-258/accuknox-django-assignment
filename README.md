# Accuknox Django Trainee Assignment

A small Django project that answers the three Django signals questions with runnable proof, plus the `Rectangle` custom class.

| Part | Where |
|---|---|
| Signal receivers | [`signals_demo/signals.py`](signals_demo/signals.py) |
| Caller side of each experiment | [`signals_demo/demos.py`](signals_demo/demos.py) |
| Management command | [`signals_demo/management/commands/signals_demo.py`](signals_demo/management/commands/signals_demo.py) |
| Signal tests | [`signals_demo/tests.py`](signals_demo/tests.py) |
| Rectangle class | [`custom_classes/rectangle.py`](custom_classes/rectangle.py) |
| Rectangle command and tests | [`custom_classes/management/commands/rectangle_demo.py`](custom_classes/management/commands/rectangle_demo.py), [`custom_classes/tests.py`](custom_classes/tests.py) |

## How to run

You need Python 3.10 or newer. The database is SQLite, so there is nothing else to install.

```bash
python -m venv venv
venv\Scripts\activate          # Windows  (on macOS/Linux: source venv/bin/activate)
pip install -r requirements.txt
python manage.py migrate
python manage.py signals_demo              # all three signal questions
python manage.py signals_demo --question 3 # just one question
python manage.py rectangle_demo            # Rectangle demo (try --length 7 --width 3)
python manage.py test                      # 6 tests
```

---

## Question 1: Are Django signals synchronous or asynchronous by default?

**Answer: synchronous.** When a model is saved, `post_save` calls every connected receiver right away, one after another, and `save()` does not return until all of them have finished. A slow receiver therefore slows down the code that called `save()`.

**Proof.** The receiver sleeps for 2 seconds. If signals were asynchronous, `save()` would return almost instantly. Instead it takes 2 seconds, and the receiver has finished before `save()` returns.

```python
# signals_demo/signals.py
@receiver(post_save, sender=SyncDemo)
def slow_receiver(sender, instance, **kwargs):
    evidence["q1_receiver_started"] = time.perf_counter()
    time.sleep(2)
    evidence["q1_receiver_finished"] = time.perf_counter()

# signals_demo/demos.py  (the caller)
caller_start = time.perf_counter()
SyncDemo.objects.create(name="q1")      # fires post_save
caller_end = time.perf_counter()
save_duration = caller_end - caller_start                 # ~2 s
finished_first = evidence["q1_receiver_finished"] <= caller_end   # True
```

## Question 2: Do Django signals run in the same thread as the caller?

**Answer: yes.** The receiver is called as a normal function call inside `send()`, so it runs in the caller's thread. No new thread is started.

**Proof.** The caller and the receiver both record `threading.get_ident()`. The two ids are identical.

```python
# signals_demo/signals.py
@receiver(post_save, sender=ThreadDemo)
def thread_receiver(sender, instance, **kwargs):
    evidence["q2_receiver_thread_id"] = threading.get_ident()

# signals_demo/demos.py  (the caller)
caller_thread_id = threading.get_ident()
ThreadDemo.objects.create(name="q2")
assert caller_thread_id == evidence["q2_receiver_thread_id"]
```

## Question 3: Do Django signals run in the same database transaction as the caller?

**Answer: yes.** Because the receiver runs synchronously in the same thread, it uses the same database connection, and so it is inside whatever transaction the caller has open. If the caller's transaction rolls back, everything the receiver wrote is rolled back too. (If you want code to run only after a successful commit, use `transaction.on_commit()`.)

**Proof.** The caller saves a `TransactionDemo` inside `transaction.atomic()`. The receiver writes a `ReceiverLog` row. Then the caller raises an exception, which rolls the transaction back. Afterwards the receiver's row is gone as well, and the receiver also reports that it was running inside an atomic block.

```python
# signals_demo/signals.py
@receiver(post_save, sender=TransactionDemo)
def transaction_receiver(sender, instance, **kwargs):
    evidence["q3_receiver_in_atomic_block"] = transaction.get_connection().in_atomic_block
    ReceiverLog.objects.create(message=f"written by receiver for {instance.name}")

# signals_demo/demos.py  (the caller)
try:
    with transaction.atomic():
        TransactionDemo.objects.create(name="q3")
        rows_inside = ReceiverLog.objects.count()   # 1: the receiver's row exists
        raise ForcedRollback()                      # roll everything back
except ForcedRollback:
    pass
rows_after = ReceiverLog.objects.count()            # 0: the receiver's row is gone too
```

### Real output of `python manage.py signals_demo`

Run on Windows 11, Python 3.14, Django 6.0.7, SQLite:

```text
Question 1: are signals synchronous or asynchronous?
  Receiver sleeps for          : 2 s
  Caller's save() took         : 2.014 s
  Receiver finished before save() returned: True
  => SYNCHRONOUS: save() was blocked until the receiver finished.

Question 2: do signals run in the caller's thread?
  Caller thread   : id=19776 name=MainThread
  Receiver thread : id=19776 name=MainThread
  => SAME THREAD: the thread ids are identical.

Question 3: do signals run in the caller's transaction?
  Receiver saw an open atomic block       : True
  Receiver rows inside the transaction    : 1
  Caller raised an exception -> transaction rolled back
  Caller rows after rollback              : 0
  Receiver rows after rollback            : 0
  => SAME TRANSACTION: the caller's rollback also undid the receiver's write.
```

(The thread id is different on every run; what matters is that the two ids match.)

---

## Custom classes in Python: `Rectangle`

Requirements:

1. Initialised with `length: int` and `width: int`.
2. Instances can be iterated over.
3. Iterating yields `{'length': <VALUE>}` first, then `{'width': <VALUE>}`.

```python
# custom_classes/rectangle.py
class Rectangle:
    def __init__(self, length: int, width: int):
        self.length = length
        self.width = width

    def __iter__(self):
        yield {'length': self.length}
        yield {'width': self.width}
```

Making `__iter__` a generator means every `for` loop gets a fresh iterator, so a rectangle can be iterated as many times as you like.

### Real output of `python manage.py rectangle_demo`

```text
Created Rectangle(length=10, width=5)
Iterating with a for loop:
  {'length': 10}
  {'width': 5}
As a list: [{'length': 10}, {'width': 5}]
```

---

## Tests

```text
> python manage.py test
Found 6 test(s).
System check identified no issues (0 silenced).
Creating test database for alias 'default'...
......
----------------------------------------------------------------------
Ran 6 tests in 2.038s

OK
Destroying test database for alias 'default'...
```

The signal tests use `TransactionTestCase` rather than `TestCase`, because `TestCase` wraps every test in its own transaction, and Question 3 needs a real transaction and a real rollback.
