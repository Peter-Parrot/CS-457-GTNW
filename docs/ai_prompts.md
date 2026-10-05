# AI Prompting Strategy
* Use role-based prompts to fine tune where the agent should specialize itself: "Assume the role of ... (network architect, python turn based game expert, etc.)"
* Adhere to test driven development processes
*

# AI Constraint Strategy
* Limit agent to local codebase neccessary for the project
* Run all agent generated code against a pytest test bench before it can be added to main.
* All AI generated code must exactly follow the defined JSON schema.
* All message headers must abide by network byte order.
* Agent shall never modify files in /docs
