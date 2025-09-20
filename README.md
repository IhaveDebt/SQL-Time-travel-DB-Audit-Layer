"""
SQL Query Performance Visualizer & Advisor
Single-file program to parse EXPLAIN plans, visualize costs, and suggest optimizations.
"""

import psycopg
import json
from pprint import pprint

DB_URL = "postgresql://postgres:postgres@localhost:5432/perf_db"

# Example table and sample data creation
def init_db():
    with psycopg.connect(DB_URL, autocommit=True) as conn:
        with conn.cursor() as cur:
            cur.execute("""
            CREATE TABLE IF NOT EXISTS products (
                id SERIAL PRIMARY KEY,
                name TEXT,
                category TEXT,
                price NUMERIC,
                stock INT
            );
            """)
            cur.execute("""
            INSERT INTO products (name, category, price, stock)
            SELECT 'Product ' || i, 'Category ' || ((i%5)+1), (i*10.0), (i*2)
            FROM generate_series(1,100) AS s(i)
            ON CONFLICT DO NOTHING;
            """)

# Parse EXPLAIN JSON plan
def parse_explain(query):
    with psycopg.connect(DB_URL) as conn:
        with conn.cursor() as cur:
            cur.execute("EXPLAIN (ANALYZE, FORMAT JSON) " + query)
            plan = cur.fetchone()[0][0]  # EXPLAIN JSON returns a list
            return plan

# Traverse plan and print human-readable info
def print_plan(plan, indent=0):
    indent_str = "  " * indent
    node_type = plan.get("Node Type")
    cost = plan.get("Total Cost")
    actual_time = plan.get("Actual Total Time")
    rows = plan.get("Plan Rows")
    print(f"{indent_str}- {node_type}: cost={cost}, rows={rows}, time={actual_time}")
    for child in plan.get("Plans", []):
        print_plan(child, indent+1)

# Suggest index if sequential scan on large table
def suggest_index(plan):
    suggestions = []
    def traverse(node):
        if node.get("Node Type") == "Seq Scan" and node.get("Relation Name"):
            rel = node.get("Relation Name")
            col = node.get("Filter")
            if col:
                suggestions.append(f"Consider index on {rel} ({col})")
        for child in node.get("Plans", []):
            traverse(child)
    traverse(plan)
    return suggestions

# CLI for demo
if __name__ == "__main__":
    init_db()
    print("SQL Query Performance Visualizer & Advisor\n")
    while True:
        query = input("Enter SQL query (or 'exit'): ")
        if query.lower() == "exit":
            break
        try:
            plan = parse_explain(query)
            print("\n--- Parsed Plan ---")
            print_plan(plan)
            suggestions = suggest_index(plan)
            if suggestions:
                print("\n--- Optimization Suggestions ---")
                for s in suggestions:
                    print(f"* {s}")
            else:
                print("\nNo obvious optimizations found.")
        except Exception as e:
            print(f"Error: {e}")
