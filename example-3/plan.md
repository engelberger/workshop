**Detailed Plan:**

**Phase 1: Setup and Configuration**

1.  **Environment Setup:**
    * Ensure Python 3 is installed.
    * Install necessary libraries:
        ```bash
        pip install requests pandas matplotlib seaborn py3Dmol # Or NGLView for potentially richer visualization
        ```
2.  **Project Structure (Suggestion):**
    ```
    interpro_afdb_demo/
    ├── run_demo.py         # Main script orchestrating the demo
    ├── config.py           # Configuration (API URLs, target protein)
    ├── api_clients.py      # Functions for interacting with InterPro & AFDB APIs
    ├── data_processor.py   # Functions for parsing and integrating data
    ├── visualizer.py       # Functions for generating 3D visualization
    ├── logs/               # Directory for log files
    ├── output/             # Directory for downloaded files & plots (optional)
    └── requirements.txt    # List of dependencies
    ```
3.  **Configuration (`config.py`):**
    * `TARGET_UNIPROT_ID = "P00520"`
    * `AFDB_API_ENDPOINT = "https://alphafold.ebi.ac.uk/api/prediction/"`
    * `INTERPRO_API_BASE = "https://www.ebi.ac.uk/interpro/api"` # Standard InterPro API base
    * `PDB_DOWNLOAD_DIR = "output/"`
    * `LOG_FILE = "logs/demo.log"`
    * `REQUEST_TIMEOUT = 30`
    * `API_DELAY = 0.2` # Short delay between API calls

4.  **Logging Setup (in `run_demo.py` or separate `logger_setup.py`):**
    * Configure the `logging` module (Level: INFO/DEBUG, FileHandler, StreamHandler, Format).
    * Log the start of the demo, including the target UniProt ID.

**Phase 2: Data Acquisition (via `api_clients.py`)**

1.  **Fetch InterPro Data:**
    * Define function `get_interpro_entries(uniprot_id)`:
        * Construct URL: `{INTERPRO_API_BASE}/protein/uniprot/{uniprot_id}/entry/interpro/` (to get InterPro domain/family entries) [cite: 11] or potentially add `/pfam/`, `/smart/` etc., if needed[cite: 9, 11].
        * **Log:** Log API call attempt.
        * Call `requests.get`, handle timeouts, check status code (200 OK), handle potential JSON errors.
        * **Log:** Log success/failure, number of entries found.
        * **Assert:** Assert response status is 200 if successful. Assert response structure looks valid (e.g., contains 'results').
        * Extract relevant data: For each entry, get accession (e.g., `IPR000744`), name, type (Domain, Family, etc.), and crucially, **protein locations** (start/end coordinates). *Note: The provided InterPro doc doesn't explicitly show locations in basic entry endpoints[cite: 9, 11]; this might require parsing deeper levels or additional API calls per entry if not directly available in the `/protein/.../entry` response. Assume for the plan that location data *can* be retrieved.* The schema description *does* mention a `residues` field containing location fragments[cite: 25]. The API call might need adjustment based on actual API behavior for location data.
        * Return a list of dictionaries, each representing a functional entry with its locations.

2.  **Fetch AlphaFold DB Data:**
    * Define function `get_afdb_data(uniprot_id)`:
        * Construct URL: `{AFDB_API_ENDPOINT}{uniprot_id}`.
        * **Log:** Log API call attempt.
        * Call `requests.get`, handle errors (timeout, status codes 404/500, JSON).
        * **Log:** Log success/failure.
        * **Assert:** Assert response status 200. Assert response is a list (usually of length 1).
        * Extract PDB URL (`pdbUrl`) and PAE Image URL (`paeImageUrl`, optional for this demo) from the first element of the response list.
        * **Assert:** Assert `pdbUrl` key exists and value looks like a URL.
        * Return a dictionary containing `{ 'pdb_url': pdb_url }`.

3.  **Download AFDB Structure File:**
    * Define function `download_structure_file(pdb_url, output_dir, uniprot_id)`:
        * Construct output path (e.g., `{output_dir}/{uniprot_id}.pdb`).
        * **Log:** Log download attempt.
        * Use `requests.get(stream=True)` to download the file, write in chunks. Handle errors.
        * **Log:** Log success/failure and file path.
        * **Assert:** Assert download was successful (e.g., check file exists and has size > 0).
        * Return the path to the downloaded file.

**Phase 3: Data Processing & Integration (via `data_processor.py`)**

1.  **Parse PDB File for pLDDT Scores:**
    * Define function `extract_plddt_from_pdb(pdb_filepath)`:
        * Input: Path to the downloaded PDB file.
        * Output: A dictionary mapping residue number (int) to pLDDT score (float).
        * **Log:** Log parsing attempt.
        * Open and read the PDB file line by line.
        * For lines starting with "ATOM" and where the atom name is "CA" (alpha carbon):
            * Extract residue sequence number (cols 23-26).
            * Extract B-factor (pLDDT score, cols 61-66).
            * Convert both to appropriate types (int, float). Handle potential `ValueError`.
            * Store in the dictionary. Ensure only one pLDDT per residue number (handle multiple chains/models if necessary, perhaps by taking the first chain 'A' or averaging if structure is multimeric - keep it simple for demo, assume chain A).
        * **Log:** Log number of residues parsed.
        * **Assert:** Assert the dictionary is not empty. Assert pLDDT values are within 0-100 range.
        * Return the residue-to-pLDDT dictionary.

2.  **Integrate InterPro Domains with pLDDT:**
    * Define function `map_domains_to_plddt(interpro_entries, plddt_data)`:
        * Input: List of InterPro entry dictionaries (from Phase 2, Step 1), residue-to-pLDDT dictionary (from Phase 3, Step 1).
        * Output: A data structure (e.g., Pandas DataFrame or list of dicts) linking residues to their pLDDT *and* the InterPro domain(s) they belong to. Also, calculate summary stats per domain.
        * Create a residue-based structure (e.g., DataFrame with columns: `residue_number`, `pLDDT`, `domain_IPR`, `domain_name`).
        * Iterate through residue numbers present in `plddt_data`.
        * For each residue, check which InterPro domain(s) it falls within based on the `locations` data in `interpro_entries`. Assign the domain accession/name. Mark residues outside any domain as 'N/A' or similar.
        * Calculate summary statistics: For each unique InterPro domain found, calculate the average pLDDT score of the residues within that domain.
        * **Log:** Log the creation of the integrated data structure and the calculated per-domain average pLDDTs.
        * **Assert:** Assert that domain start/end coordinates were valid and mapped correctly. Assert average pLDDTs were calculated.
        * Return both the detailed residue mapping and the per-domain summary statistics.

**Phase 4: Visualization (The Demo Core) (via `visualizer.py`)**

1.  **Visualize Structure with Domain & Confidence:**
    * Define function `display_structure_with_domains(pdb_filepath, residue_domain_plddt_map, domain_avg_plddt)`:
        * Input: Path to PDB file, the detailed residue map, and the per-domain average pLDDTs.
        * Use `py3Dmol` (or `NGLView`):
            * Load the PDB model (`view.addModel(open(pdb_filepath, 'r').read(), 'pdb')`).
            * Set the default style, coloring by pLDDT using the B-factor property: `view.setStyle({'cartoon': {'colorscheme': {'prop':'b','gradient': 'roygb','min': 50,'max': 90}}})`. This replicates standard AFDB coloring.
            * **Highlight Domains:** Iterate through the unique domains found in `residue_domain_plddt_map`. For each domain:
                * Create a selection (`resi` list) containing all residue numbers belonging to that domain.
                * Apply a distinct style to that selection to overlay the domain information. Ideas:
                    * Change representation: `view.setStyle({'resi': resi_list}, {'cartoon': {'style': 'trace', 'color': 'white'}}) # Example: White trace`
                    * Add a surface representation: `view.addSurface(py3Dmol.VDW, {'opacity':0.7,'colorscheme':{'prop':'b','gradient': 'roygb','min': 50,'max': 90}}, {'resi': resi_list})`
                    * Add labels (might clutter): `view.addResLabels({'resi': resi_list[0]}, {'backgroundColor': 'lightgray', 'fontColor': 'black'})` (Label start?)
                * *(Choose one clear highlighting method for the demo)*.
            * Zoom to the structure (`view.zoomTo()`).
            * **Log:** Log the generation of the 3D view.
            * Display the view (`view.show()` or return the view object).
            * **Print Summary:** Print the calculated average pLDDT scores for each displayed domain clearly below the visualization.

**Phase 5: Orchestration and Demo Narrative (in `run_demo.py`)**

1.  **Main Script Logic:**
    * Setup logging.
    * Log start.
    * Call `get_interpro_entries` for `config.TARGET_UNIPROT_ID`. Handle potential failure.
    * Call `get_afdb_data` for `config.TARGET_UNIPROT_ID`. Handle potential failure.
    * Call `download_structure_file` using the retrieved `pdb_url`. Handle potential failure.
    * Call `extract_plddt_from_pdb` using the downloaded file path. Handle potential failure.
    * Call `map_domains_to_plddt` using InterPro data and pLDDT data. Handle potential failure.
    * **Demo Output:**
        * Print a clear title (e.g., "Demo: Visualizing AlphaFold Confidence in InterPro Domains for {UniProt ID}").
        * Print the list of identified InterPro domains and their average pLDDT scores.
        * Call `display_structure_with_domains` to show the interactive 3D structure.
    * Log completion of the demo.

2.  **Demo Narrative Points (To be spoken or displayed alongside):**
    * "We're analyzing protein {UniProt ID} ({Protein Name})."
    * "First, we used the InterPro API to find its known functional domains: [List domains found]."
    * "Next, we fetched the predicted 3D structure and per-residue confidence scores (pLDDT) from the AlphaFold Database API."
    * "Here is the structure, colored by AlphaFold's confidence (blue=high, red=low)."
    * "Now, we've highlighted the regions corresponding to the InterPro domains [point to highlights]."
    * "We can see that Domain X is predicted with an average confidence of {score}%, while Domain Y has a lower confidence of {score}%."
    * "This tells us how much we might trust the predicted structure in functionally important regions, guiding further research or modeling efforts."
