Detailed Explanation and Requirements
markdown
Copy code
# Kaplan-Meier and GSEA Analysis Pipeline

This script provides a comprehensive workflow for analyzing clinical and gene expression data, generating Kaplan-Meier survival curves, and performing Gene Set Enrichment Analysis (GSEA). Below is a detailed explanation of the script's functionality, along with the necessary requirements.

## Requirements

To run this script, you need to have the following Python libraries installed:

- `pandas`
- `matplotlib`
- `numpy`
- `lifelines`
- `gseapy`

You can install these libraries using pip:

```sh
pip install pandas matplotlib numpy lifelines gseapy
Script Overview
1. Data Loading and Validation
load_data(filepath)
Reads a CSV file from the given filepath into a Pandas DataFrame.

python
Copy code
def load_data(filepath):
    return pd.read_csv(filepath)
ensure_columns_present(data, required_columns)
Checks if the required columns are present in the DataFrame. Raises a ValueError if any columns are missing.

python
Copy code
def ensure_columns_present(data, required_columns):
    if not all(col in data.columns for col in required_columns):
        raise ValueError(f"Missing one or more required columns: {required_columns}")
2. Data Preprocessing
convert_os_status(data)
Converts the 'OS_STATUS' column to an 'event' column with binary values. '1
' is converted to 1 (event), and other values are converted to 0 (no event).

python
Copy code
def convert_os_status(data):
    data['event'] = data['OS_STATUS'].apply(lambda x: 1 if x == '1:DECEASED' else 0)
    return data
limit_months(data, max_months)
Filters the DataFrame to include only rows where 'OS_MONTHS' is less than or equal to the specified max_months.

python
Copy code
def limit_months(data, max_months):
    return data[data['OS_MONTHS'] <= max_months]
create_expression_groups(data, gene_expression)
Creates binary columns indicating high and low gene expression based on the top and bottom quartiles.

python
Copy code
def create_expression_groups(data, gene_expression):
    top_quartile_threshold = data[gene_expression].quantile(0.75)
    bottom_quartile_threshold = data[gene_expression].quantile(0.25)
    data.loc[:, 'high_expression'] = data.loc[:, gene_expression] >= top_quartile_threshold
    data.loc[:, 'low_expression'] = data.loc[:, gene_expression] <= bottom_quartile_threshold
    return data
3. GSEA Analysis
run_gsea(data, gsea_filepath, gene_set_filepath, subtype)
Performs GSEA using the prerank function from the gseapy library. It reads the GSEA data from an Excel file and ranks the genes based on their log2 ratios.

python
Copy code
def run_gsea(data, gsea_filepath, gene_set_filepath, subtype):
    gsea_data = pd.read_excel(gsea_filepath, sheet_name=subtype)
    if 'Log2 Ratio' not in gsea_data.columns:
        raise ValueError("The GSEA file is missing the 'Log2 Ratio' column.")
    gsea_data['Log2 Ratio'] = pd.to_numeric(gsea_data['Log2 Ratio'], errors='coerce')
    rnk = gsea_data[['Gene', 'Log2 Ratio']].sort_values(by='Log2 Ratio', ascending=False).dropna()
    gsea_results = prerank(rnk=rnk, gene_sets=gene_set_filepath, threads=4, permutation_num=100)
    return gsea_results
4. Plotting Results
plot_gsea_results(gsea_results, ax)
Plots the GSEA results, highlighting significant pathways.

python
Copy code
def plot_gsea_results(gsea_results, ax):
    results_df = gsea_results.res2d
    results_df['NES'] = pd.to_numeric(results_df['NES'], errors='coerce')
    results_df = results_df[results_df['FDR q-val'] < 0.05]

    high_expr_terms = results_df[results_df['NES'] > 0].nlargest(10, 'NES')
    low_expr_terms = results_df[results_df['NES'] < 0].nsmallest(10, 'NES')

    high_expr_terms['NES'] = high_expr_terms['NES'].abs()
    low_expr_terms['NES'] = low_expr_terms['NES'].abs()

    terms = pd.concat([high_expr_terms, low_expr_terms])
    terms.sort_values('NES', ascending=True, inplace=True)

    y_pos = np.arange(len(terms))
    colors = ['red' if term in high_expr_terms['Term'].values else 'blue' for term in terms['Term']]
    ax.barh(y_pos, terms['NES'], color=colors, edgecolor='black')
    ax.set_yticks(y_pos)
    ax.set_yticklabels(terms['Term'])
    ax.set_xlabel('-log10(P-value)')
    ax.invert_yaxis()

    for i, (nes, pval) in enumerate(zip(terms['NES'], terms['NOM p-val'])):
        ax.text(nes, i, f"P={pval:.1e}", va='center', ha='left' if nes > 0 else 'right')
plot_kaplan_meier(data, gene_expression, gsea_filepath, gene_set_filepath, subtype)
Plots Kaplan-Meier survival curves and calls run_gsea to plot GSEA results.

python
Copy code
def plot_kaplan_meier(data, gene_expression, gsea_filepath, gene_set_filepath, subtype):
    kmf = KaplanMeierFitter()
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 6))

    high_expression_group = data[data['high_expression']]
    low_expression_group = data[data['low_expression']]

    if not high_expression_group.empty and not low_expression_group.empty:
        kmf.fit(durations=high_expression_group['OS_MONTHS'], event_observed=high_expression_group['event'], label='Top Quartile')
        kmf.plot_survival_function(ax=ax1, ci_show=True, color='red')

        kmf.fit(durations=low_expression_group['OS_MONTHS'], event_observed=low_expression_group['event'], label='Bottom Quartile')
        kmf.plot_survival_function(ax=ax1, ci_show=True, color='blue')

        # Perform log-rank test
        results = logrank_test(high_expression_group['OS_MONTHS'], low_expression_group['OS_MONTHS'],
                               event_observed_A=high_expression_group['event'], event_observed_B=low_expression_group['event'])
        p_value = results.p_value

        ax1.text(0.05, 0.05, f'Log-rank p-value: {p_value:.4f}', transform=ax1.transAxes, fontsize=10, bbox=dict(facecolor='white', alpha=0.5))

    ax1.set_title(f'Kaplan-Meier Survival Curve for {gene_expression} Expression in {subtype}')
    ax1.set_xlabel('Months')
    ax1.set_ylabel('Survival Probability')
    ax1.legend()

    gsea_results = run_gsea(data, gsea_filepath, gene_set_filepath, subtype)
    plot_gsea_results(gsea_results, ax2)

    plt.tight_layout()
    plt.show()
5. Main Function
main()
Coordinates the entire workflow, including loading data, preprocessing, filtering by subtype, and plotting results.

python
Copy code
def main():
    data_filepath = 'cleaned_clinical_nsd1_data.csv'
    gsea_filepath = '_NSD1_high_vs_low_quartiles.xlsx'
    gene_set_filepath = 'h.all.v2023.2.Hs.symbols.gmt'

    data = load_data(data_filepath)
    ensure_columns_present(data, ['OS_MONTHS', 'OS_STATUS', 'CLAUDIN_SUBTYPE', 'NSD1'])
    data = convert_os_status(data)
    data = limit_months(data, 200)
    data = create_expression_groups(data, 'NSD1')

    print("Enter the number corresponding to the subtype: 1 for LumA, 2 for LumB, 3 for Basal:")
    subtype_choice = int(input().strip())
    subtype_map = {1: 'LumA', 2: 'LumB', 3: 'Basal'}
    chosen_subtype = subtype_map.get(subtype_choice, 'LumA')
    
    data = data[data['CLAUDIN_SUBTYPE'] == chosen_subtype]
    
    if data.empty:
        print(f"No data available for the selected subtype: {chosen_subtype}")
        return

    plot_kaplan_meier(data, 'NSD1', gsea_filepath, gene_set_filepath, chosen_subtype)

if __name__ == '__main__':
    main()
Notes
Dependencies: Ensure all required libraries are installed.
User Input: The script prompts the user for subtype selection.
Error Handling: Additional error handling can be included as needed.
Data Files: Ensure file paths are correct for your environment.
Testing: Test with sample data to ensure the script works as expected.