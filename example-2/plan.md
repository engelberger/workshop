**Goal:** Download PDB files for 100 specified UniProt accessions from the AlphaFold DB API, extract basic statistics from these files, and generate plots summarizing these statistics.

**Core Principles:**

1.  **Simplicity (No Over-engineering):** Use standard libraries and straightforward logic. Avoid complex frameworks or design patterns where simple functions suffice.
2.  **Modularity:** Break down the process into distinct functions (e.g., fetch data, download file, parse PDB, plot stats).
3.  **Robustness:** Implement error handling for API calls, file operations, and data parsing.
4.  **Detailed Logging:** Log key actions, successes, failures, and data points to trace execution flow and diagnose issues.
5.  **Data Validation:** Use assertions to check critical assumptions about the data at various stages.
6.  **Reproducibility:** Ensure the script can be run reliably given the same inputs.

---

**Detailed Plan:**

**Phase 1: Setup and Configuration**

1.  **Environment Setup:**
    * Ensure Python 3 is installed.
    * Install necessary libraries:
        ```bash
        pip install requests pandas matplotlib seaborn py3Dmol # Add Biopython if needed later for complex parsing
        ```
    * *(Self-Correction: Initially forgot pandas/matplotlib, added them for stats/plotting)*

2.  **Project Structure (Suggestion):**
    ```
    afdb_analyzer/
    ├── main.py             # Main script orchestrating the process
    ├── config.py           # Configuration variables (API URL, output dirs)
    ├── uniprot_ids.txt     # Input file with 100 UniProt IDs, one per line
    ├── logs/               # Directory for log files
    ├── downloaded_pdbs/    # Directory to store downloaded PDB files
    ├── plots/              # Directory to store generated plots
    └── requirements.txt    # List of dependencies
    ```

3.  **Configuration (`config.py`):**
    * Define constants:
        * `AFDB_API_ENDPOINT = "https://alphafold.ebi.ac.uk/api/prediction/"`
        * `UNIPROT_ID_FILE = "uniprot_ids.txt"`
        * `PDB_OUTPUT_DIR = "downloaded_pdbs/"`
        * `PLOT_OUTPUT_DIR = "plots/"`
        * `LOG_FILE = "logs/afdb_analyzer.log"`
        * `REQUEST_TIMEOUT = 30` # Seconds
        * `API_DELAY = 0.5` # Seconds to wait between API calls to be polite

4.  **Logging Setup (in `main.py` or a dedicated `utils.py`):**
    * Import the `logging` module.
    * Configure logging:
        * Set level (e.g., `logging.INFO` or `logging.DEBUG` for more detail).
        * Use `logging.FileHandler` to write to `config.LOG_FILE`.
        * Use `logging.StreamHandler` to print logs to the console.
        * Define a clear log format (e.g., `%(asctime)s - %(levelname)s - %(module)s - %(message)s`).
    * Log the start of the script execution.

**Phase 2: Data Acquisition**

1.  **Input Preparation:**
    * Create the `uniprot_ids.txt` file. Populate it with 100 valid UniProt accession IDs, one per line.
    * *(Assumption: User will provide this list)*.

2.  **Read UniProt IDs (in `main.py`):**
    * Define a function `load_uniprot_ids(filepath)`:
        * Reads IDs from `config.UNIPROT_ID_FILE`.
        * Handles `FileNotFoundError`.
        * Removes whitespace and filters out empty lines.
        * **Log:** Log the number of IDs loaded.
        * **Assert:** Assert that the loaded list is not empty.
        * Returns a list of IDs.

3.  **Fetch API Data (in `main.py` or `api_handler.py`):**
    * Define a function `Workspace_prediction_data(uniprot_id)`:
        * Input: Single UniProt ID string.
        * Construct the full API URL using `config.AFDB_API_ENDPOINT`.
        * **Log:** Log the attempt to fetch data for the ID.
        * Use `requests.get(url, timeout=config.REQUEST_TIMEOUT)`.
        * Include error handling:
            * Catch `requests.exceptions.RequestException` (timeout, connection error). Log the specific error. Return `None`.
            * Check `response.status_code`.
                * If 200: Log success. Parse JSON (`response.json()`). Handle potential `JSONDecodeError`. Return the parsed data (likely a list containing one dictionary).
                * If 404: Log "UniProt ID not found in AFDB". Return `None`.
                * Other non-200 codes: Log the status code and error message (`response.text`). Return `None`.
        * **Assert:** Assert the input `uniprot_id` is a non-empty string.
        * Returns the JSON data (list) or `None` on failure.

4.  **Download PDB File (in `main.py` or `file_handler.py`):**
    * Define a function `download_pdb(pdb_url, output_path)`:
        * Input: PDB URL string, destination file path string.
        * **Log:** Log the attempt to download from `pdb_url` to `output_path`.
        * Use `requests.get(pdb_url, timeout=config.REQUEST_TIMEOUT, stream=True)` *(Use stream=True for potentially large files)*.
        * Include error handling similar to `Workspace_prediction_data` (RequestException, status codes).
        * If successful (status code 200):
            * Open `output_path` in binary write mode (`'wb'`).
            * Iterate through `response.iter_content(chunk_size=8192)` and write chunks to the file.
            * **Log:** Log successful download and file save location.
            * Return `True`.
        * If failed: Log the error. Return `False`.
        * **Assert:** Assert `pdb_url` looks like a URL (e.g., starts with 'http'). Assert `output_path` is a non-empty string.

5.  **Orchestration Loop (in `main.py`):**
    * Load UniProt IDs using `load_uniprot_ids`.
    * Create output directories (`config.PDB_OUTPUT_DIR`, `config.PLOT_OUTPUT_DIR`, `logs/`) if they don't exist (`os.makedirs(..., exist_ok=True)`).
    * Initialize counters for successes and failures.
    * Initialize a list to store metadata (like PDB path, ID, maybe pLDDT later) for successfully downloaded files.
    * Iterate through the loaded UniProt IDs:
        * **Log:** Log processing status (e.g., "Processing ID X/100: [UniProt_ID]").
        * Call `Workspace_prediction_data(uniprot_id)`.
        * **Wait:** `time.sleep(config.API_DELAY)` *before* the next API call.
        * If data is fetched successfully:
            * **Assert:** Check if the response is a list and not empty (`assert isinstance(data, list) and data`).
            * Extract the PDB URL. Based on the example, it's likely `data[0]['pdbUrl']`. Add checks: does the dictionary exist? Does the 'pdbUrl' key exist? Is the value a string?
            * If PDB URL is found:
                * Construct the `output_pdb_path` (e.g., `os.path.join(config.PDB_OUTPUT_DIR, f"{uniprot_id}.pdb")`).
                * Call `download_pdb(pdb_url, output_pdb_path)`.
                * If download is successful:
                    * Increment success counter.
                    * Append relevant info (like `uniprot_id`, `output_pdb_path`) to the metadata list.
                * Else (download failed): Increment failure counter.
            * Else (PDB URL not found in response): Log the issue. Increment failure counter.
        * Else (API fetch failed): Increment failure counter.
    * **Log:** Log the final count of successful downloads and failures.

**Phase 3: Data Processing and Statistics**

1.  **Define Statistics:** Decide on simple, relevant statistics. Examples:
    * Number of Residues per protein.
    * Average pLDDT score per protein (requires parsing the B-factor column).
    * *(Keep it simple first, e.g., just residue count)*.

2.  **Parse PDB for Statistics (in `main.py` or `parser.py`):**
    * Define a function `extract_stats_from_pdb(pdb_filepath)`:
        * Input: Path to a downloaded PDB file.
        * Output: A dictionary containing calculated statistics (e.g., `{'residue_count': N, 'avg_plddt': X}`).
        * **Log:** Log the attempt to parse the file.
        * Initialize variables (e.g., `residue_set = set()`, `plddt_scores = []`).
        * Open and read the PDB file line by line (`try-except FileNotFoundError, IOError`).
        * Iterate through lines:
            * If line starts with "ATOM":
                * **(Residue Count):** Extract the residue sequence number (columns 23-26) and chain ID (column 22). Add `(chain_id, res_seq)` to `residue_set`. *(This correctly counts residues even with multiple atoms per residue)*.
                * **(pLDDT):** Extract the B-factor (columns 61-66). Try converting it to float (`try-except ValueError`). If successful and the atom name is 'CA' (alpha carbon, columns 13-16), append the score to `plddt_scores`.
        * Calculate final statistics:
            * `residue_count = len(residue_set)`
            * `avg_plddt = sum(plddt_scores) / len(plddt_scores)` if `plddt_scores` is not empty, else `None` or `0`.
        * **Log:** Log the extracted stats for the file.
        * **Assert:** Assert `residue_count >= 0`. If pLDDT was extracted, assert scores are within a reasonable range (e.g., 0-100).
        * Return the dictionary of stats. Handle errors during parsing gracefully (e.g., return `None` or default values and log the error).

3.  **Process All Downloaded PDBs (in `main.py`):**
    * Initialize an empty list `all_stats`.
    * Iterate through the `metadata` list (containing info about successfully downloaded files):
        * Get the `pdb_filepath` and `uniprot_id`.
        * Call `extract_stats_from_pdb(pdb_filepath)`.
        * If stats are extracted successfully:
            * Add the `uniprot_id` to the stats dictionary.
            * Append the stats dictionary to `all_stats`.
        * Else: Log failure to parse the specific PDB file.
    * Convert `all_stats` list of dictionaries into a Pandas DataFrame (`stats_df = pd.DataFrame(all_stats)`).
    * **Log:** Log the dimensions of the resulting DataFrame.
    * **Assert:** Assert the DataFrame is not empty (if any PDBs were processed successfully). Assert expected columns exist.
    * **(Optional):** Save the DataFrame to a CSV file (`stats_df.to_csv(...)`) for later use.

**Phase 4: Visualization**

1.  **Plotting Function(s) (in `main.py` or `plotter.py`):**
    * Import `matplotlib.pyplot as plt` and `seaborn as sns`.
    * Define function(s) like `plot_statistics(stats_df)`:
        * Input: The Pandas DataFrame containing the statistics.
        * Generate plots using `matplotlib`/`seaborn`:
            * **Example 1:** Histogram of Residue Counts (`sns.histplot(data=stats_df, x='residue_count')`). Add title, labels. Save figure (`plt.savefig(os.path.join(config.PLOT_OUTPUT_DIR, 'residue_count_histogram.png'))`). Clear the plot (`plt.clf()`).
            * **Example 2:** Histogram or Boxplot of Average pLDDT Scores (`sns.histplot(data=stats_df, x='avg_plddt', bins=20)` or `sns.boxplot(data=stats_df, y='avg_plddt')`). Add title, labels. Save figure. Clear plot.
        * **Log:** Log the generation and saving of each plot.
        * **Assert:** Assert the input is a Pandas DataFrame and contains the necessary columns for plotting.

2.  **Call Plotting Function (in `main.py`):**
    * After creating `stats_df`, call `plot_statistics(stats_df)`.

**Phase 5: Finalization and Testing**

1.  **Code Review:** Read through the code. Check for clarity, comments, adherence to the plan, and robustness.
2.  **Testing (Manual/Integration):**
    * Run the script with a *small* list of UniProt IDs (e.g., 3-5) including one known to work and potentially one known to fail (if possible).
    * Check the log file (`afdb_analyzer.log`) for expected messages, errors, and stats.
    * Verify that the correct PDB files are downloaded to `downloaded_pdbs/`.
    * Inspect the generated plots in `plots/` for reasonableness.
    * *(Formal unit tests using `unittest` or `pytest` could be added but might lean towards over-engineering for this scope. Good logging and assertions provide significant validation).*
3.  **Refinement:** Adjust code based on testing results (e.g., improve error messages, handle edge cases discovered).
4.  **Documentation:** Add docstrings to functions explaining what they do, their parameters, and what they return. Add comments for complex logic sections. Add a README explaining how to set up and run the script.
