# Beaver-choice-paper-multi-agent-system
Beaver's Choice Paper Company - Multi-Agent System
Overview
A multi-agent system that automates paper supply order management.

Agents
Orchestrator Agent - Routes and coordinates requests
Inventory Agent - Checks stock and triggers reorders
Quoting Agent - Generates quotes with bulk discounts
Sales Agent - Finalizes transactions
Tech Stack
Framework: smolagents + LiteLLM
Model: gpt-4o-mini
Database: SQLite3
Language: Python 3
Discount Policy
Small order: 0%
Medium order: 5%
Large order: 15%
Files
project_starter.py - Main implementation
test_results.csv - Evaluation results

