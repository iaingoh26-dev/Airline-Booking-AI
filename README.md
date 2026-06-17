# ✈️ FlightAI — Airline Ticket Price Assistant

An AI-powered customer support chatbot for a fictional airline called **FlightAI**.
Built using **OpenAI GPT**, **SQLite**, and **Gradio**.

---

## 🚀 What It Does

- Answers customer questions about ticket prices
- Uses a **real database** to store and look up prices
- Powered by **ChatGPT** with custom tools
- Runs a **chat UI** in your browser via Gradio

---

---

## 🧠 How It Works

### 1. Initialization
Loads the API key and connects to OpenAI's GPT model.

```python
load_dotenv(override=True)
MODEL = "gpt-4.1-mini"
openai = OpenAI()
```

---

### 2. AI Personality
Gives the AI its role and rules.

```python
system_message = """
You are a helpful assistant for an Airline called FlightAI.
Give short, courteous answers, no more than 1 sentence.
Always be accurate. If you don't know the answer, say so.
"""
```

---

### 3. Database Setup
Creates a local SQLite database to store ticket prices permanently.

```python
DB = "prices.db"
with sqlite3.connect(DB) as conn:
    cursor = conn.cursor()
    cursor.execute('CREATE TABLE IF NOT EXISTS prices (city TEXT PRIMARY KEY, price REAL)')
    conn.commit()
```

---

### 4. Seeding Initial Prices
Adds starting prices for 5 cities into the database.

```python
ticket_prices = {"london": 799, "paris": 899, "tokyo": 1420, "sydney": 2999, "berlin": 499}
for city, price in ticket_prices.items():
    set_ticket_price(city, price)
```

| City   | Price  |
|--------|--------|
| London | $799   |
| Paris  | $899   |
| Tokyo  | $1420  |
| Sydney | $2999  |
| Berlin | $499   |

---

### 5. Tool Functions
Functions that look up and save ticket prices from the database.

```python
# Look up a price
def get_ticket_prices(city):
    with sqlite3.connect(DB) as conn:
        cursor = conn.cursor()
        cursor.execute('SELECT price FROM prices WHERE city = ?', (city.lower(),))
        result = cursor.fetchone()
        return f"Ticket price to {city} is ${result[0]}" if result else "No price data available"

# Save a price
def set_ticket_price(city, price):
    with sqlite3.connect(DB) as conn:
        cursor = conn.cursor()
        cursor.execute(
            'INSERT INTO prices (city, price) VALUES (?, ?) ON CONFLICT(city) DO UPDATE SET price = ?',
            (city.lower(), price, price)
        )
        conn.commit()
```

---

### 6. Telling the AI About the Tool
Describes the function to OpenAI so the AI knows when and how to use it.

```python
price_function = {
    "name": "get_ticket_prices",
    "description": "Get the price of a return ticket to the destination city.",
    "parameters": {
        "type": "object",
        "properties": {
            "destination_city": {
                "type": "string",
                "description": "The city that the customer wants to travel to",
            },
        },
        "required": ["destination_city"],
        "additionalProperties": False
    }
}

tools = [{"type": "function", "function": price_function}]
```

---

### 7. Handling Tool Calls
Runs the function when the AI requests it and returns the result.

```python
def handle_tool_calls(message):
    responses = []
    for tool_call in message.tool_calls:
        if tool_call.function.name == "get_ticket_prices":
            arguments = json.loads(tool_call.function.arguments)
            city = arguments.get('destination_city')
            price_details = get_ticket_prices(city)
            responses.append({
                "role": "tool",
                "content": price_details,
                "tool_call_id": tool_call.id
            })
    return responses
```

---

### 8. Main Chat Function
The brain of the chatbot — sends messages, handles tool calls, and returns answers.

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    while response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        responses = handle_tool_calls(message)
        messages.append(message)
        messages.extend(responses)
        response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    return response.choices[0].message.content
```

<img width="638" height="640" alt="image" src="https://github.com/user-attachments/assets/bc4b56c7-51cd-481d-9fef-91ae96f94466" />

