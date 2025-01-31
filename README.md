# SQLite-LangChain-Bot-AI-Powered-SQL-Query-Generator-
This project integrates LangChain with Azure OpenAI to generate and execute SQL queries dynamically on a SQLite database based on user instructions. It automates data management, ensuring updates without duplicates.

# Create database file
import sqlite3

# SQLite database file
DB_PATH = "path to db"  # New database file

# Connect to the new database (or create it if it doesn't exist)
conn = sqlite3.connect(DB_PATH)
cursor = conn.cursor()

# Drop existing tables if they exist
cursor.execute("DROP TABLE IF EXISTS table1")
cursor.execute("DROP TABLE IF EXISTS table2")
cursor.execute("DROP TABLE IF EXISTS table3")

# Create the tables 
cursor.execute("""
CREATE TABLE IF NOT EXISTS table1 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    age INTEGER NOT NULL
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS table2 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    customer_id INTEGER,
    amount REAL NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS table3 (
    ID INT PRIMARY KEY,
    Month VARCHAR(20),
    Sales DECIMAL(10, 2),
    Profit DECIMAL(10, 2),
    Comments VARCHAR(25)
)
""")

# Insert 10 rows into table1
table1 = [("Alice", 25), ("Bob", 30), ("Charlie", 22), ("David", 28), ("Emma", 35),
             ("Frank", 40), ("Grace", 29), ("Hannah", 26), ("Ian", 32), ("Jack", 27)]
cursor.executemany("INSERT INTO table1 (name, age) VALUES (?, ?)", table1)

# Insert 10 rows into table2
table2 = [(1, 100.50), (2, 200.75), (3, 150.20), (4, 80.99), (5, 220.30),
          (6, 90.50), (7, 120.40), (8, 300.00), (9, 50.25), (10, 175.60)]
cursor.executemany("INSERT INTO table2 (customer_id, amount) VALUES (?, ?)", table2)

# Insert 12 rows into table3
cursor.executemany("""
INSERT INTO table3 (ID, Month, Sales, Profit, Comments) VALUES (?, ?, ?, ?, ?)
""", [
    (1, 'January', 1000.00, 200.00,''),
    (2, 'February', 1500.00, 300.00,''),
    (3, 'March', 2000.00, 400.00,''),
    (4, 'April', 2500.00, 500.00,''),
    (5, 'May', 3000.00, 600.00,''),
    (6, 'June', 3500.00, 700.00,''),
    (7, 'July', 4000.00, 800.00,''),
    (8, 'August', 4500.00, 900.00,''),
    (9, 'September', 5000.00, 1000.00,''),
    (10, 'October', 5500.00, 1100.00,''),
    (11, 'November', 6000.00, 1200.00,''),
    (12, 'December', 6500.00, 1300.00,'')
])

# Commit the changes and close the connection
conn.commit()
conn.close()

print("New database created with the same tables and data as the old one.")


#Main.py
import sqlite3
import os
from langchain_openai import AzureChatOpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain

# SQLite database file
DB_PATH = "path to db"

# Azure OpenAI credentials
llm = AzureChatOpenAI(
    azure_endpoint="",
    model="",
    api_version="",
    api_key="",
    temperature=0,
    max_tokens=None,
)

# Read instructions from file
with open("path to text", "r") as file:    #path to text file containing instructions
    instructions = file.read()

# SQL generation prompt
prompt_template = PromptTemplate(
    input_variables=["instructions"],
    template="""Given the following instructions, generate valid SQL queries for a SQLite database:

Instructions:
{instructions}

The database schema is as follows:
1. table1 (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, age INTEGER)
2. table2 (id INTEGER PRIMARY KEY AUTOINCREMENT, customer_id INTEGER, amount REAL, FOREIGN KEY(customer_id) REFERENCES customers(id))
3. table3 (ID INT PRIMARY KEY, Month TEXT, Sales DECIMAL(10, 2), Profit DECIMAL(10, 2), Comments TEXT)

#Give prompt according to your needs
"""
)

# Generate SQL query
chain = LLMChain(llm=llm, prompt=prompt_template)
sql_query = chain.run(instructions).strip()

# Debugging: Print the generated SQL
print("\nGenerated SQL Query:\n", sql_query)

# Remove unwanted markdown formatting (if present)
sql_query = sql_query.replace("```sql", "").replace("```", "").strip()

# Connect to SQLite
connection = sqlite3.connect(DB_PATH)
cursor = connection.cursor()

# Function to check if a column exists
def column_exists(table, column):
    cursor.execute(f"PRAGMA table_info({table})")
    return any(row[1] == column for row in cursor.fetchall())

# Function to remove duplicates
def remove_duplicates():
    print("\nRemoving duplicates...")

    cursor.execute("""
        DELETE FROM customers WHERE id NOT IN (
            SELECT MIN(id) FROM customers GROUP BY name, age
        )
    """)

    cursor.execute("""
        DELETE FROM orders WHERE id NOT IN (
            SELECT MIN(id) FROM orders GROUP BY customer_id, amount
        )
    """)

    cursor.execute("""
        DELETE FROM SalesData WHERE ID NOT IN (
            SELECT MIN(ID) FROM SalesData GROUP BY Month, Sales, Profit, Comments
        )
    """)

    connection.commit()
    print("Duplicates removed.")

# Remove duplicates before inserting new updates
remove_duplicates()

# Execute each SQL statement separately
try:
    for query in sql_query.split(";"):
        query = query.strip()
        if query:
            if "ALTER TABLE" in query and "ADD COLUMN" in query:
                # Extract table and column name from the query
                words = query.split()
                table_name = words[2]  # 3rd word should be table name
                column_name = words[5]  # 6th word should be column name

                # Check if column already exists
                if column_exists(table_name, column_name):
                    print(f"\nSkipping column addition: '{column_name}' already exists in '{table_name}'.")
                    continue

            cursor.execute(query)

    connection.commit()
    print("\nDatabase updated successfully.")
except Exception as e:
    print("\nError executing SQL:", e)
finally:
    cursor.close()
    connection.close()

# View updated database contents
def view_table(table_name):
    connection = sqlite3.connect(DB_PATH)
    cursor = connection.cursor()

    try:
        cursor.execute(f"SELECT * FROM {table_name}")
        rows = cursor.fetchall()
        print(f"\nContents of {table_name}:")
        if rows:
            for row in rows:
                print(row)
        else:
            print("No data found.")
    except Exception as e:
        print("Error:", e)
    finally:
        cursor.close()
        connection.close()

# Show all tables after update
for table in ["table1", "table2", "table3"]:
    view_table(table)
