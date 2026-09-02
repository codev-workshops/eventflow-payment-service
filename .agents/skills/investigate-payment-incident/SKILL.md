---
name: investigate-payment-incident
description: Investigate a payment-processing incident in eventflow-payment-service, specifically a currency-specific failure (e.g. JPY orders raising an unhandled ValueError), and capture it with a regression test.
---

# Investigate a payment-processing incident

## Purpose

Use this skill when the payment service is failing for orders in a specific
currency (the known case is JPY, a zero-decimal currency). It walks the
investigator through the code path in reading order, gives an exact
reproduction taken from the source, and describes how to add a regression
test using the repository's existing test conventions.

Every statement below is grounded in files in this repository. Where a value
could not be confirmed from source, it is marked "verify".

Scope: this skill covers investigation and test authoring only. Fixing
`app/processor.py` is out of scope.

## Investigation reading order

### 1. `app/consumer.py` — where the failure surfaces

- `_process_message(message_body)` does `json.loads`, builds
  `OrderCreatedEvent(**event_dict)`, then calls
  `process_order_payment(event.data)` and stores the returned
  `PaymentRecord` in the in-memory `payments` dict.
- The `except ValueError` branch logs
  `"Payment processing failed — unhandled validation error"`, tries
  `_update_order_status(event.data.order_id, "failed")` (a `PATCH` to
  `{order_service_url}/api/orders/{order_id}/status`, skipped when
  `ORDER_SERVICE_URL` is unset), and then **re-raises**.
- `_consumer_loop` wraps each message in `try/except Exception`; on the
  re-raised error it logs `"Failed to process message — abandoning"` and
  calls `receiver.abandon_message(message)`. So the symptom in logs is the
  pair of messages above, and the message is returned to the queue rather
  than completed.

### 2. `app/processor.py` — where the defect lives

- `convert_to_display_amount(amount_minor, currency)` unconditionally
  returns `amount_minor / 100`; `currency` is not consulted. The docstring and
  inline comment both label this as the bug.
- `process_order_payment(event_data: OrderEventData) -> PaymentRecord`
  calls `convert_to_display_amount`, then
  `process_payment_through_gateway(display_amount, currency, order_id)`.
- `process_payment_through_gateway` calls
  `validate_payment_amount(display_amount, currency)` before returning a
  simulated successful `GatewayResponse`.
- `validate_payment_amount` looks up
  `MINIMUM_TRANSACTION_THRESHOLDS.get(currency, 0.50)` and raises
  `ValueError(f"Amount {display_amount} {currency} is below minimum threshold {threshold} {currency}")`
  when `display_amount < threshold`.
- The threshold map, verbatim from `app/processor.py`:

  ```python
  MINIMUM_TRANSACTION_THRESHOLDS: dict[str, float] = {
      "USD": 0.50,
      "EUR": 0.50,
      "GBP": 0.30,
      "JPY": 500.0,
      "KRW": 500.0,
      "CHF": 0.50,
      "CAD": 0.50,
      "AUD": 0.50,
      "CNY": 3.00,
      "INR": 50.0,
  }
  ```

- Note: `process_order_payment` only ever returns
  `PaymentStatus.FAILED` when `gateway_response.success` is false. The
  gateway simulation always returns `success=True`, so the currency failure
  is never turned into a `FAILED` record — it escapes as a `ValueError`.

### 3. `app/config.py` — surrounding wiring (rule out environment)

`Settings(BaseSettings)` with `model_config = {"env_file": ".env", ...}`
defines:

- `azure_servicebus_connection_string: str = ""`
- `azure_servicebus_queue_name: str = "order-events"`
- `applicationinsights_connection_string: str = ""`
- `order_service_url: str = ""`
- `log_level`, `environment`, `service_name`, `service_version`

Check these to rule out a misconfigured queue or callback. The threshold map
is **not** in `app/config.py`; it lives in `app/processor.py`.

### Supporting models (`app/models.py`)

```python
class PaymentStatus(str, Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"

class OrderCreatedEvent(BaseModel):
    event_id: str
    event_type: str
    timestamp: datetime
    data: "OrderEventData"

class OrderEventData(BaseModel):
    order_id: str
    customer_id: str
    currency: str
    amount: int
    items: list[OrderItem]   # OrderItem: product_id, name, quantity, unit_price (int)

class PaymentRecord(BaseModel):
    payment_id: str            # default uuid4
    order_id: str
    customer_id: str
    currency: str
    amount_minor: int
    amount_display: float
    status: PaymentStatus = PaymentStatus.PENDING
    processed_at: datetime     # default now(UTC)
    error_message: str | None = None
```

## Reproducing the currency-specific failure

Input (from the module docstring and comments in `app/processor.py`):
an `OrderEventData` with `currency="JPY"` and `amount=15800`, wrapped in an
`OrderCreatedEvent` when arriving via Service Bus.

Flow:

1. `convert_to_display_amount(15800, "JPY")` → `15800 / 100 = 158.0`
2. `validate_payment_amount(158.0, "JPY")` → threshold is `500.0`
   (`MINIMUM_TRANSACTION_THRESHOLDS["JPY"]`)
3. `158.0 < 500.0` → `ValueError("Amount 158.0 JPY is below minimum threshold 500.0 JPY")`
4. `_process_message` logs and re-raises; `_consumer_loop` abandons the message.

The inline comment chain in `process_payment_through_gateway` states this
directly: `JPY 15800 → display_amount = 158.00 → below 500 JPY threshold → CRASH`.

Minimal in-process reproduction (no Service Bus needed):

```python
from app.models import OrderEventData
from app.processor import process_order_payment

process_order_payment(
    OrderEventData(
        order_id="order-jpy-001",
        customer_id="cust-jpy",
        currency="JPY",
        amount=15800,
        items=[{"product_id": "p1", "name": "Item", "quantity": 1, "unit_price": 15800}],
    )
)
# raises ValueError on current main
```

## Test conventions

- `pyproject.toml` declares:

  ```toml
  [tool.pytest.ini_options]
  asyncio_mode = "auto"
  testpaths = ["tests"]
  ```

- A `tests/` directory **does exist** (`tests/__init__.py`,
  `tests/conftest.py`, `tests/test_processor.py`). Mirror its style:
  plain `pytest` classes (`TestConvertToDisplayAmount`,
  `TestProcessOrderPayment`, `TestHealthEndpoints`), fixtures
  `usd_order_event_data` / `eur_order_event_data` / `client` in
  `conftest.py`, and `OrderEventData` built with `items` as plain dicts.
- `tests/test_processor.py` states in its docstring that it only covers USD
  and EUR and that the JPY/KRW zero-decimal bug is not covered, "which is
  why it passes CI but fails in production". README ("The Bug") says the
  same: CI tests pass because they only cover USD/EUR.
- Add new tests under `tests/` (e.g. extend `tests/test_processor.py` or add
  `tests/test_zero_decimal.py`).

## Adding a regression test

Frame the tests as capturing the bug. They **fail on current `main`** and
only pass once `convert_to_display_amount` is fixed (out of scope here).

1. Parametrized conversion test over `(currency, amount_minor, expected_display)`:

   ```python
   import pytest
   from app.processor import convert_to_display_amount

   @pytest.mark.parametrize(
       ("currency", "amount_minor", "expected_display"),
       [("USD", 15800, 158.00), ("JPY", 15800, 15800.0)],
   )
   def test_convert_to_display_amount(currency, amount_minor, expected_display):
       assert convert_to_display_amount(amount_minor, currency) == expected_display
   ```

2. Validation-boundary test: `validate_payment_amount` with the *correct*
   JPY display amount must not raise, while a genuinely sub-threshold amount
   must:

   ```python
   from app.processor import validate_payment_amount

   def test_jpy_at_or_above_threshold_does_not_raise():
       validate_payment_amount(15800.0, "JPY")

   def test_jpy_below_threshold_raises():
       with pytest.raises(ValueError):
           validate_payment_amount(499.0, "JPY")
   ```

3. Integration test through `process_order_payment`:

   ```python
   from app.models import OrderEventData, PaymentStatus
   from app.processor import process_order_payment

   def test_process_jpy_order_completes():
       event_data = OrderEventData(
           order_id="order-jpy-001",
           customer_id="cust-jpy",
           currency="JPY",
           amount=15800,
           items=[{"product_id": "p1", "name": "Item", "quantity": 1, "unit_price": 15800}],
       )
       payment = process_order_payment(event_data)
       assert payment.status == PaymentStatus.COMPLETED
       assert payment.amount_minor == 15800
       assert payment.amount_display == 15800.0
   ```

## Repository commands

Copied from repository evidence; do not add flags not shown here.

| Step | Command | Source |
|---|---|---|
| Install | `poetry install` | README "Local Development" |
| Run tests | `poetry run pytest -v` | README "Local Development" (CI uses `poetry run pytest -v --tb=short`) |
| Lint | `poetry run ruff check app/ tests/` | `.github/workflows/ci.yml` "Lint with Ruff" step |

Ruff configuration in `pyproject.toml` (dev dependency `ruff = "^0.2.0"`):

```toml
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]
```

README does not document a lint command in prose; the lint command above is
taken from the CI workflow, which is the only place it appears. Confirm it
still matches `.github/workflows/ci.yml` before relying on it.

Verify: Python 3.11 is required (`python = "^3.11"`, CI uses 3.11); the
system default may differ locally.
