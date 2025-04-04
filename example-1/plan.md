**Goal:** Download 100 Protein Data Bank (PDB) files, parse them to extract basic structural information, calculate summary statistics, and generate plots visualizing these statistics.

**Guiding Principles:**

1.  **Simplicity (No Over-engineering):** Start with straightforward functions and scripts. Avoid complex class hierarchies or design patterns unless they significantly simplify the core logic.
2.  **Best Practices:** Use virtual environments, manage dependencies, write modular code, use meaningful names, and include basic error handling.
3.  **Detailed Logging:** Implement comprehensive logging to track the execution flow, decisions, successes, and failures at different levels (DEBUG, INFO, WARNING, ERROR).
4.  **Data Validation (Assertions):** Use `assert` statements strategically to check assumptions about the data being processed (e.g., file downloaded successfully, expected keys exist in parsed data).

**Technology Stack:**

* **Language:** Python 3.x
* **Core Libraries:**
    * `requests`: For downloading files via HTTP.
    * `biopython`: For parsing PDB files (`Bio.PDB`).
    * `pandas`: For data manipulation and basic statistics.
    * `matplotlib` / `seaborn`: For plotting.
* **Standard Libraries:**
    * `os`: For path manipulation.
    * `logging`: For application logging.
    * `random` / `PDBList` (from `biopython`): For selecting PDB IDs.
    * `datetime`: For timestamping logs.

**Project Structure:**

```
pdb_analyzer/
├── src/
│   ├── __init__.py
│   ├── config.py           # Configuration variables (paths, constants)
│   ├── downloader.py       # Functions for downloading PDB files
│   ├── parser.py           # Functions for parsing PDB files
│   ├── analysis.py         # Functions for calculating statistics
│   ├── plotter.py          # Functions for generating plots
│   └── main.py             # Main script orchestrating the workflow
├── data/                   # Directory to store downloaded PDB files
│   └── pdb/
├── logs/                   # Directory to store log files
│   └── app.log
├── plots/                  # Directory to store generated plots
├── tests/                  # (Optional but recommended) Unit/Integration tests
│   ├── __init__.py
│   ├── test_downloader.py
│   └── ...
├── requirements.txt        # Project dependencies
└── README.md               # Project description
```

**Detailed Plan:**

**Phase 1: Setup & Configuration**

1.  **Environment Setup:**
    * Create a virtual environment: `python -m venv venv`
    * Activate it: `source venv/bin/activate` (Linux/macOS) or `.\venv\Scripts\activate` (Windows)
2.  **Install Dependencies:**
    * Create `requirements.txt` with:
        ```
        requests
        biopython
        pandas
        matplotlib
        seaborn
        ```
    * Install: `pip install -r requirements.txt`
3.  **Project Structure:** Create the directories outlined above.
4.  **Configuration (`src/config.py`):**
    * Define constants:
        * `NUM_PDB_TO_DOWNLOAD = 100`
        * `PDB_DOWNLOAD_URL_TEMPLATE = "https://files.rcsb.org/download/{}.pdb"`
        * `DATA_DIR = os.path.abspath("../data/pdb")`
        * `LOG_FILE = os.path.abspath("../logs/app.log")`
        * `PLOT_DIR = os.path.abspath("../plots")`
        * `LOG_LEVEL = logging.DEBUG` # Set to DEBUG for detailed info, INFO for production
5.  **Logging Setup (`src/main.py` or a dedicated `logging_config.py`):**
    * Configure the `logging` module:
        * Set level (from `config.py`).
        * Create a `FileHandler` to write logs to `config.LOG_FILE`.
        * Create a `StreamHandler` to print logs to the console.
        * Define a log format (e.g., `%(asctime)s - %(name)s - %(levelname)s - %(message)s`).
    * Get logger instances in each module (`logger = logging.getLogger(__name__)`).

**Phase 2: PDB ID Acquisition**

1.  **Strategy:** Get a list of 100 valid PDB IDs. A simple approach is to use `biopython`'s `PDBList`.
2.  **Implementation (`src/downloader.py`):**
    * Create a function `get_pdb_ids(count: int) -> list[str]`.
    * Inside, use `PDBList().get_all_entries()` which returns a list of all current PDB IDs.
    * Log: `INFO: Fetching list of all available PDB IDs.`
    * Select `count` IDs from this list (e.g., the first `count`, or a random sample).
    * Log: `INFO: Selected {count} PDB IDs for download.`
    * Return the list of IDs.

**Phase 3: PDB File Downloading (`src/downloader.py`)**

1.  **Function:** Create `download_pdb_file(pdb_id: str, download_dir: str) -> str | None`.
2.  **Logic:**
    * Construct the URL using `config.PDB_DOWNLOAD_URL_TEMPLATE`.
    * Define the output file path (e.g., `os.path.join(download_dir, f"{pdb_id}.pdb")`).
    * Log: `DEBUG: Attempting to download PDB ID: {pdb_id} from {url} to {output_path}`.
    * Use `requests.get(url, stream=True)` to download.
    * **Error Handling:** Check `response.raise_for_status()` for HTTP errors (4xx, 5xx). Catch `requests.exceptions.RequestException`.
    * If successful:
        * Write the content to the output file.
        * Log: `INFO: Successfully downloaded {pdb_id}.pdb`.
        * **Assertion:** `assert os.path.exists(output_path)` (Verify file was created).
        * **Assertion:** `assert os.path.getsize(output_path) > 0` (Verify file is not empty).
        * Return the `output_path`.
    * If failed:
        * Log: `ERROR: Failed to download {pdb_id}. Reason: {error_message}`.
        * Return `None`.
3.  **Orchestration (`src/main.py`):**
    * Call `get_pdb_ids`.
    * Loop through the IDs.
    * Call `download_pdb_file` for each ID.
    * Keep track of successfully downloaded file paths.
    * Log progress (e.g., `INFO: Downloaded {i+1}/{total} PDB files.`).

**Phase 4: PDB Parsing & Data Extraction (`src/parser.py`)**

1.  **Function:** Create `parse_pdb_file(pdb_filepath: str) -> dict | None`.
2.  **Logic:**
    * Log: `DEBUG: Attempting to parse PDB file: {pdb_filepath}`.
    * Initialize `PDBParser`: `parser = PDBParser(QUIET=True)` (suppress warnings during parsing).
    * Use a `try...except` block to handle potential parsing errors (e.g., `ValueError`, file format issues).
    * Parse the file: `structure = parser.get_structure(pdb_id, pdb_filepath)` (extract `pdb_id` from filename).
    * **Assertion:** `assert structure is not None`, `assert structure.id == pdb_id`.
    * Extract desired information:
        * Resolution (from header): `resolution = structure.header.get('resolution', None)`
        * Number of models: `num_models = len(structure)`
        * Number of chains: `num_chains = sum(1 for _ in structure.get_chains())`
        * Total number of residues: `num_residues = sum(1 for _ in structure.get_residues())`
        * Total number of atoms: `num_atoms = sum(1 for _ in structure.get_atoms())`
        * *(Optional: Add more complex metrics like residue types count, secondary structure if needed)*
    * Store extracted data in a dictionary.
    * **Assertion:** Check if key metrics were extractable (e.g., `assert 'num_chains' in extracted_data`). Be mindful that some data like resolution might be missing in some PDB files, so handle `None` gracefully downstream.
    * Log: `INFO: Successfully parsed {pdb_filepath}. Resolution: {resolution}, Chains: {num_chains}, Residues: {num_residues}`.
    * Return the dictionary.
    * If parsing fails:
        * Log: `ERROR: Failed to parse {pdb_filepath}. Reason: {error_message}`.
        * Return `None`.
3.  **Orchestration (`src/main.py`):**
    * Loop through the successfully downloaded file paths.
    * Call `parse_pdb_file` for each.
    * Collect the resulting dictionaries into a list.

**Phase 5: Statistics Calculation (`src/analysis.py`)**

1.  **Function:** Create `calculate_statistics(parsed_data_list: list[dict]) -> pd.DataFrame`.
2.  **Logic:**
    * Log: `INFO: Calculating statistics for {len(parsed_data_list)} parsed PDB structures.`
    * Convert the list of dictionaries into a `pandas` DataFrame: `df = pd.DataFrame(parsed_data_list)`.
    * **Assertion:** `assert not df.empty` (if parsing yielded any results).
    * **Assertion:** Check expected columns exist: `assert all(col in df.columns for col in ['pdb_id', 'resolution', 'num_chains', ...])`.
    * Handle missing data (e.g., fill `NaN` in resolution if necessary for certain calculations, or just ignore them in specific stats).
    * Calculate summary statistics using DataFrame methods:
        * `df.describe()` for basic stats (mean, median, std dev, min, max) on numerical columns (resolution, chains, residues, atoms).
        * Value counts for categorical data if added (e.g., molecule type).
    * Log: `INFO: Calculated summary statistics:\n{df.describe().to_string()}`.
    * *(Optional: Calculate correlations or more advanced stats if needed).*
    * Return the DataFrame (it contains both raw extracted data and can be used for plotting).

**Phase 6: Plotting (`src/plotter.py`)**

1.  **Functions:** Create functions for specific plots, e.g.:
    * `plot_histogram(data: pd.Series, title: str, xlabel: str, filename: str)`
    * `plot_scatter(df: pd.DataFrame, x_col: str, y_col: str, title: str, filename: str)`
2.  **Logic:**
    * Use `matplotlib.pyplot` and/or `seaborn`.
    * Take the `pandas` DataFrame or Series as input.
    * Clean data for plotting (e.g., drop rows with `NaN` for the plotted variable).
    * Generate plots:
        * Histogram of resolutions (handle potential `None`/`NaN` values).
        * Histogram of the number of chains per structure.
        * Histogram of the number of residues per structure.
        * Scatter plot (e.g., resolution vs. number of residues).
    * Set clear titles, labels.
    * Save plots to the `config.PLOT_DIR` using `plt.savefig()`. Ensure the directory exists.
    * Log: `INFO: Generated plot: {title} and saved to {filename}`.
    * Use `plt.clf()` or `plt.close()` after saving each plot to prevent plots from overlapping in memory.
3.  **Orchestration (`src/main.py`):**
    * Call the plotting functions with the results DataFrame.

**Phase 7: Main Script & Execution (`src/main.py`)**

1.  **Import necessary modules and functions.**
2.  **Setup Logging:** Call the logging configuration function.
3.  **Create Directories:** Ensure `config.DATA_DIR`, `config.LOG_FILE`'s directory, and `config.PLOT_DIR` exist (`os.makedirs(..., exist_ok=True)`).
4.  **Log Start:** `logging.info("Starting PDB analysis workflow.")`
5.  **Execute Workflow:**
    * Get PDB IDs (`downloader.get_pdb_ids`).
    * Download files (`downloader.download_pdb_file` in a loop). Store successful paths.
    * Parse downloaded files (`parser.parse_pdb_file` in a loop). Store successful results.
    * Calculate statistics (`analysis.calculate_statistics`).
    * Generate plots (`plotter.plot_histogram`, etc.).
6.  **Log End:** `logging.info("PDB analysis workflow finished successfully.")`
7.  **Error Handling:** Wrap major phases (downloading, parsing) in `try...except` blocks to catch unexpected errors and log them, allowing the script to potentially continue or exit gracefully.

**Testing Considerations:**

* **Unit Tests (`tests/`):**
    * Test `downloader.py`: Mock `requests.get` to simulate successful/failed downloads without network calls.
    * Test `parser.py`: Use a few known, small PDB files stored locally to test parsing logic and data extraction accuracy. Test edge cases (missing header info, unusual structures).
    * Test `analysis.py`: Create sample DataFrames to verify statistics calculations.
* **Integration Test:** Run `main.py` with `NUM_PDB_TO_DOWNLOAD` set to a small number (e.g., 3-5) to test the end-to-end flow.
