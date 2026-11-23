Step 1: Build Your Scraper Server (starter_server.py)
Open starter_server.py. This file will become your custom MCP server that scrapes websites. We need to complete the two functions marked as tools.

Task 1.1: Complete the scrape_websites tool
This function will use Firecrawl to scrape a list of websites, save their content to files, and update a central scraped_metadata.json file.

Inside the scrape_websites function, replace the # continue your solution here ... comment with logic to do the following:

Load Existing Metadata: Try to open and load scraped_metadata.json. If it doesn't exist or is empty, initialize scraped_metadata as an empty dictionary {}.
Initialize: Create a list called successful_scrapes to hold the names of providers we successfully scrape.
Loop Through Websites: Iterate over the websites dictionary (which contains provider_name -> url pairs).
Inside the Loop (for each website):
Wrap the scraping logic in a try...except block to handle any errors.
Log that you are starting to scrape (e.g., logger.info(f"Scraping {provider_name}: {url}")).
Call Firecrawl: scrape_result = app.scrape(url, formats=formats).model_dump().
Create a metadata dictionary for this provider, including its name, URL, domain, and scrape time.
If Successful: app.scrape() raises an exception when a scrape fails (your except block handles it), so here check that scrape_result holds content for the formats you asked for (for example, scrape_result.get('markdown')).
Save Content: Loop through the formats (e.g., 'markdown', 'html'). For each format, get the content from scrape_result, create a filename (like f"{provider_name}_{format_type}.txt"), and write the content to a file in the SCRAPE_DIR.
Update Metadata: Add the filenames to your metadata dictionary (under a content_files key). Add the page title and description from the scrape result's metadata (for example, scrape_result['metadata']['title']).
Add the provider_name to your successful_scrapes list.
If Failed: If scrape_result holds no content, log that for this provider.
Finally: Add your metadata dictionary to the main scraped_metadata dictionary (e.g., scraped_metadata[provider_name] = metadata).
After the Loop:
Write the entire scraped_metadata dictionary back to the scraped_metadata.json file.
Log the final results.
Return the successful_scrapes list.
Task 1.2: Complete the extract_scraped_info tool
This function will read the data you just saved.

Inside the extract_scraped_info function, replace the # contine your response here ... comment with logic to do the following:

Load Metadata: Open and load the scraped_metadata.json file inside a try...except block.
Find a Match: Loop through the scraped_metadata dictionary. For each provider_name and metadata entry, check if your identifier matches any of these:
The provider_name
The metadata.get('url', '')
The metadata.get('domain', '')
If a Match is Found:
Make a copy of the metadata (e.g., result = metadata.copy()).
Check if the metadata has 'content_files'. If it does, create a new result['content'] dictionary.
Loop through the content_files (e.g., format_type, filename).
Read the content from the file (e.g., os.path.join(SCRAPE_DIR, filename)) and store it in result['content'][format_type].
Return the result dictionary as a formatted JSON string: return json.dumps(result, indent=2).
If No Match/Error: If the loop finishes with no match or if you hit an error (like FileNotFoundError), return a string message like f"There's no saved information related to identifier '{identifier}'.".
Step 2: Build Your Client Application (starter_client.py)
Open starter_client.py. This is your "manager" app that will talk to your server. We just need to fill in the sections marked # complete.

In Configuration.load_config:
Implement the logic to open and read the file_path.
Load the JSON: config = json.load(f).
Add error handling: wrap the file read in a try...except block.
Check if mcpServers is in the config: if "mcpServers" not in config: ...
Return the config.
In Server.initialize:
The server_params object is empty. Fill it using the self.config dictionary:
server_params = StdioServerParameters(
command=command,
args=self.config["args"],
env={**os.environ,** self.config["env"]} if self.config.get("env") else None,
)
In Server.list_tools:
Check if the session exists: if not self.session: ...
Call the session to get tools: tools_response = await self.session.list_tools().
Format the response: loop over tools_response.tools, build a tool_def for each tool as below, collect them in a list, and return the list:
tool_def: ToolDefinition = {
"name": tool.name,
"description": tool.description,
"input_schema": tool.inputSchema
}
In Server.execute_tool:
Implement the retry loop. The core of the try block is:
logging.info(f"Executing {tool_name}...")
result = await self.session.call_tool(name=tool_name, arguments=arguments, read_timeout_seconds=timedelta(seconds=60))
return result
In DataExtractor.extract_and_store_data:
After loading the pricing_data, loop through the plans.
Before the loop, add a helper that doubles single quotes, because each value sits inside single quotes in the SQL (deepinfra's becomes deepinfra''s):
def sql_text(value):
    return str(value).replace("'", "''")
Inside the for plan in pricing_data.get("plans", []): loop, add the sqlite_server.execute_tool call to insert the data into the database, passing each text value through sql_text:
await self.sqlite_server.execute_tool("write_query", {
"query": f"""
INSERT INTO pricing_plans (company_name, plan_name, input_tokens, output_tokens, currency, billing_period, features, limitations, source_query)
VALUES (
'{sql_text(pricing_data.get("company_name", "Unknown"))}',
'{sql_text(plan.get("plan_name", "Unknown Plan"))}',
'{plan.get("input_tokens", 0)}',
'{plan.get("output_tokens", 0)}',
'{sql_text(plan.get("currency", "USD"))}',
'{sql_text(plan.get("billing_period", "unknown"))}',
'{sql_text(json.dumps(plan.get("features", [])))}',
'{sql_text(plan.get("limitations", ""))}',
'{sql_text(user_query)}')
"""
})
In ChatSession.process_query:
Model Name: Change <ENTER_MODEL_NAME> to a Vocareum-available model. Use the same dated model the starter already uses in _get_structured_extraction: 'claude-sonnet-4-5-20250929'.
Next to assistant_content = [] (inside the while loop), add tool_results = [].
if content.type == 'text': block:
Add the text to the full_response and show it: full_response += content.text + "\n", then print(content.text).
Add the content to the assistant's message: assistant_content.append(content).
elif content.type == 'tool_use': block:
This is the full tool-use loop. Steps 1 to 5 run for each tool the model asks for; steps 6 and 7 run once, after the for loop:
Append the tool use request to assistant_content.
Get the tool id, input (its arguments), and name from the content.
Find the server that has this tool.
Run the tool on that server: result = await server.execute_tool(content.name, content.input).
Add the tool_result to tool_results: {"type": "tool_result", "tool_use_id": content.id, "content": result.content}.
If tool_results has results, append the assistant's message ({'role': 'assistant', 'content': assistant_content}) and then one user message holding every result ({'role': 'user', 'content': tool_results}) to messages. A response can ask for several tools at once, and all of their results go in that one message. Then call self.anthropic.messages.create again with the same model, max_tokens and tools and the new messages list, and store it in response. The while loop then handles the new response.
If tool_results is empty, the response was text only, so the model is done: process_query = False.
In ChatSession.show_stored_data:
Add the tool call to read from the pricing_plans table:
pricing = await self.sqlite_server.execute_tool("read_query", {
    "query": "SELECT company_name, plan_name, input_tokens, output_tokens, currency FROM pricing_plans ORDER BY created_at DESC LIMIT 5"
})

print("\nRecently Stored Data:")
print("=" * 50)

print("\nPricing Plans:")
# read_query returns the rows as text (a Python list of dicts), so parse it first.
import ast
rows = ast.literal_eval(pricing.content[0].text)
for plan in rows:
    print(f"  • {plan['company_name']}: {plan['plan_name']} - Input Token ${plan['input_tokens']}, Output Tokens ${plan['output_tokens']}")

print("=" * 50)
Step 3: Run and Test Your Project
You are now ready to run your application!

Run the Client: In your terminal (with your virtual environment activated), run:
python starter_client.py
Test Your Tools: Your client will start, connect to all 3 servers (your scraper, sqlite, and filesystem), and give you a Query: prompt.
Use the Test Prompts from the README.md:
First, scrape the sites:
"scrape these sites: {'cloudrift': 'https://www.cloudrift.ai/inference', 'deepinfra': 'https://deepinfra.com/pricing', 'fireworks': 'https://fireworks.ai/pricing#serverless-pricing', 'groq': 'https://groq.com/pricing'}"
Then, ask questions about the scraped content:
"How much does cloudrift ai (https://www.cloudrift.ai/inference) charge for deepseek v3?"
"How much does deepinfra (https://deepinfra.com/pricing) charge for deepseek v3"
"Compare cloudrift ai and deepinfra's costs for deepseek v3"
Check your database:
Type show data to see the structured data your DataExtractor saved to the SQLite database.
Note: One suggestion would be to read the data from the SQLite database at the time of comparing the costs of Cloudrift AI and DeepInfra, and then execute the comparison prompt on the retrieved data. Otherwise, the agent can scrape the sites again, which may result in additional API credit consumption.



Project Rubric

On the following pages, you'll find the rubric that will be used to assess your project, please check the rubric while completing the project, and before submission.

Your submission will consist of two parts: your completed code and proof that your project works.

Step 1: Gather Your Evidence
Before you clean your project folder, you must run your completed application and capture screenshots of it working. This is the most important part of your submission.

Run Your Client: In your terminal (with your virtual environment activated), run your client:
python starter_client.py
Execute the Test Prompts: At the Query: prompt, you will run the test prompts from the README.md file and take screenshots.
Test 1: Scraping Run the scrape command. Make sure your screenshot shows the command you typed and the successful "Successfully scraped 4 out of 4 websites" (or similar) output.
Query: scrape these sites: {'cloudrift': 'https://www.cloudrift.ai/inference', 'deepinfra': 'https://deepinfra.com/pricing', 'fireworks': 'https://fireworks.ai/pricing#serverless-pricing', 'groq': 'https://groq.com/pricing'}
Take Screenshot 1
Test 2: Asking a Question Run one of the comparison questions. Your screenshot should show the query and the bot's full, natural-language answer.
Query: Compare cloudrift ai and deepinfra's costs for deepseek v3
Take Screenshot 2
Test 3: Checking the Database Run the show data command. This proves your DataExtractor worked and saved the information to the SQLite database. Your screenshot must show the Query: show data command and the formatted table of pricing plans it prints out.
Query: show data
Take Screenshot 3
Create Your Evidence File:
Create a single document (e.g., using Google Docs, Word, or a Markdown file) named evidence.
Paste your three screenshots into this document in order.
Save this file in your project folder as evidence.pdf or evidence.md.
Step 2: Clean Your Project Folder
Now that you have your evidence, you must clean your project folder to remove temporary files, cached files, and personal secrets.

Please DELETE the following files and folders from your project directory:

.env (This file contains your secret API keys and should NEVER be shared.)
test.db (This database file is generated automatically.)
scraped_content/ (This entire folder and all the text files inside it.)
.venv/ (Your virtual environment folder.)
Any __pycache__/ folders you see.
Step 3: Check for Required Files
After cleaning, your project folder should contain all the code and evidence needed for the reviewer. Please make sure the following files are present:

evidence.pdf (or evidence.md)
starter_server.py (Your completed server file)
starter_client.py (Your completed client file using the Anthropic API)
server_config.json (Make sure the "llm_inference" path points to starter_server.py!)
pyproject.toml
uv.lock
README.md
Step 4: Create Your ZIP File
You will submit a single .zip file containing all the required project files.

Navigate to the folder outside your project directory.
Right-click on your project folder (the one containing starter_server.py, evidence.pdf, etc.).
Choose "Compress..." (on macOS) or "Send to > Compressed (zipped) folder" (on Windows).
Name the resulting zip file mcp_project_submission.zip.
Step 5: Final Check
To prevent submission issues, a great final step is to check the file you just created:

Move your new .zip file to a different location (like your Desktop).
Unzip it (double-click on macOS or "Extract All..." on Windows).
Look inside the new folder. Do you see all the required files from Step 3, especially your evidence.pdf file?
Are all the files you were supposed to delete in Step 2 gone?
If everything looks correct, you are ready to submit.

Step 6: Submit Your Project
Upload the single mcp_project_submission.zip file you created and checked. You're all done!
