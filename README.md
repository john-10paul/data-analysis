# data-analysis
python data analysis of DV Lottery from 2016-2018
# ============================================
# IMPORT PACKAGES
# ============================================
import os
import re
import sys
import pdfplumber
import pandas as pd
import numpy as np

# Configure pandas display(Purpose: Tells pandas to show ALL data, not hide any rows or columns)
pd.set_option('display.max_rows', None)
pd.set_option('display.max_columns', None)
pd.set_option('display.width', None)
pd.set_option('display.max_colwidth', 50)

# ============================================
# CONFIGURATION - CHANGE THIS PATH
# ============================================
pdf_path = r"C:\Users\WHO\Downloads\DV AES statistics by FSC 2016-2018.pdf"

print("="*80)
print("📊 DV LOTTERY DATA EXTRACTOR 2016-2018")
print("="*80)

# ============================================
# STEP 1: CHECK IF FILE EXISTS
# ============================================
if not os.path.exists(pdf_path):
    print(f"\n❌ ERROR: File not found at: {pdf_path}")
    print("\nPlease check:")
    print("1. The file path is correct")
    print("2. The file hasn't been moved or renamed")
    print("3. You're using the correct folder name")
    sys.exit(1)

file_size = os.path.getsize(pdf_path)
print(f"\n✅ File found: {os.path.basename(pdf_path)}")
print(f"   📁 Size: {file_size:,} bytes ({file_size/1024/1024:.2f} MB)")

# ============================================
# STEP 2: DIAGNOSTIC - SEE WHAT'S IN THE PDF
# ============================================
print("\n" + "🔍"*40)
print("🔍 STEP 1: DIAGNOSING PDF STRUCTURE")
print("🔍"*40)

with pdfplumber.open(pdf_path) as pdf:
    print(f"\n📄 PDF has {len(pdf.pages)} pages")
    
    # Look at first page in detail
    first_page = pdf.pages[0]
    
    # Check for text
    text = first_page.extract_text()
    if text:
        print(f"\n📝 TEXT EXTRACTED (first 500 characters):")
        print("-"*60)
        print(text[:500])
        print("-"*60)
    else:
        print("\n❌ NO TEXT EXTRACTED - PDF may be scanned images")
    
    # Check for tables with different strategies
    print("\n📊 TESTING TABLE EXTRACTION STRATEGIES:")
    
    strategies = [
        ("Default", {}),
        ("Lines", {'vertical_strategy': 'lines', 'horizontal_strategy': 'lines'}),
        ("Text", {'vertical_strategy': 'text', 'horizontal_strategy': 'text'}),
    ]
    
    for strategy_name, settings in strategies:
        tables = first_page.extract_tables(settings)
        print(f"   {strategy_name}: {len(tables)} tables found")
        
        if tables and len(tables) > 0:
            first_table = tables[0]
            print(f"      First table: {len(first_table)} rows × {len(first_table[0]) if first_table[0] else 0} columns")
    
    # Search for Kenya
    print("\n🔍 SEARCHING FOR 'KENYA' IN FIRST 5 PAGES:")
    kenya_found = False
    for page_num in range(min(5, len(pdf.pages))):
        page = pdf.pages[page_num]
        page_text = page.extract_text()
        if page_text and 'Kenya' in page_text:
            print(f"   ✅ Page {page_num+1}: Found 'Kenya'")
            kenya_found = True
            
            # Show the line containing Kenya
            lines = page_text.split('\n')
            for line in lines:
                if 'Kenya' in line:
                    print(f"      Line: {line}")
                    break
    
    if not kenya_found:
        print("   ❌ 'Kenya' not found in first 5 pages")

# ============================================
# STEP 3: EXTRACT ALL DATA
# ============================================
print("\n" + "📥"*40)
print("📥 STEP 2: EXTRACTING ALL DATA")
print("📥"*40)

all_data = []

with pdfplumber.open(pdf_path) as pdf:
    for page_num, page in enumerate(pdf.pages, 1):
        print(f"\n📄 Processing page {page_num}...")
        
        # Try multiple extraction strategies
        tables = []
        
        # Strategy 1: Default
        tables = page.extract_tables()
        
        # Strategy 2: If no tables, try with lines
        if not tables or len(tables) == 0:
            tables = page.extract_tables({
                'vertical_strategy': 'lines',
                'horizontal_strategy': 'lines'
            })
        
        print(f"   Found {len(tables)} table(s)")
        
        for table_idx, table in enumerate(tables, 1):
            if not table or len(table) < 2:
                continue
            
            # Convert to DataFrame
            df = pd.DataFrame(table)
            
            # Clean up obvious non-data rows
            df = df[~df[0].astype(str).str.contains('DIVERSITY|Foreign State|Page|Grand Totals|www|Electronic|website|rowspan|FY|Entrants|Derivatives|Total', 
                                                     case=False, na=False)]
            
            df = df.dropna(how='all')
            
            if len(df) > 0:
                all_data.append(df)
                print(f"   ✅ Table {table_idx}: Added {len(df)} rows")

# ============================================
# STEP 4: COMBINE ALL DATA
# ============================================
print("\n" + "🔗"*40)
print("🔗 STEP 3: COMBINING DATA")
print("🔗"*40)

if not all_data:
    print("\n❌ No data extracted from PDF!")
    print("\nTroubleshooting steps:")
    print("1. Run the diagnostic section above to see what's in the PDF")
    print("2. If PDF is scanned, you'll need OCR (pytesseract)")
    print("3. If PDF has complex tables, try tabula-py: pip install tabula-py")
    sys.exit(1)

df_combined = pd.concat(all_data, ignore_index=True)
print(f"\n✅ Combined {len(all_data)} tables")
print(f"   Total rows: {df_combined.shape[0]:,}")
print(f"   Total columns: {df_combined.shape[1]}")

# ============================================
# STEP 5: AUTO-DETECT COLUMN STRUCTURE
# ============================================
print("\n" + "🔎"*40)
print("🔎 STEP 4: ANALYZING COLUMN STRUCTURE")
print("🔎"*40)

# Rename first column as Country
df_combined = df_combined.rename(columns={0: 'Country'})

# Clean country names
df_combined['Country'] = df_combined['Country'].astype(str).str.strip()
df_combined = df_combined[df_combined['Country'].notna()]
df_combined = df_combined[df_combined['Country'] != '']
df_combined = df_combined[~df_combined['Country'].str.contains('Page|Table|Continued', case=False, na=False)]

print(f"\n📊 After cleaning: {len(df_combined)} rows")

# Look at remaining columns
print(f"\n📋 Column structure:")
for i, col in enumerate(df_combined.columns):
    sample_val = df_combined[col].iloc[0] if len(df_combined) > 0 else 'N/A'
    print(f"   Column {i}: '{col}' (sample: {str(sample_val)[:30]})")

# Try to identify numeric columns
numeric_cols = []
for col in df_combined.columns[1:]:  # Skip Country column
    # Check if this column contains numbers
    sample = df_combined[col].astype(str).str.replace(',', '').str.replace('N/A', '0')
    try:
        pd.to_numeric(sample, errors='coerce')
        numeric_cols.append(col)
    except:
        pass

print(f"\n🔢 Detected {len(numeric_cols)} potential numeric columns")

# ============================================
# STEP 6: SMART COLUMN MAPPING
# ============================================
print("\n" + "🧩"*40)
print("🧩 STEP 5: MAPPING COLUMNS")
print("🧩"*40)

# Based on the number of numeric columns, choose the right mapping
num_numeric = len(numeric_cols)

if num_numeric == 9:
    print("✅ Detected 9 numeric columns - using 3-year mapping")
    column_mapping = {
        1: 'FY2016_Entrants',
        2: 'FY2016_Derivatives', 
        3: 'FY2016_Total',
        4: 'FY2017_Entrants',
        5: 'FY2017_Derivatives',
        6: 'FY2017_Total',
        7: 'FY2018_Entrants',
        8: 'FY2018_Derivatives',
        9: 'FY2018_Total'
    }
elif num_numeric == 6:
    print("⚠️ Detected 6 numeric columns - using 2-year mapping")
    column_mapping = {
        1: 'FY2016_Entrants',
        2: 'FY2016_Derivatives', 
        3: 'FY2016_Total',
        4: 'FY2017_Entrants',
        5: 'FY2017_Derivatives',
        6: 'FY2017_Total',
    }
elif num_numeric == 3:
    print("⚠️ Detected 3 numeric columns - using totals-only mapping")
    column_mapping = {
        1: 'FY2016_Total',
        2: 'FY2017_Total',
        3: 'FY2018_Total'
    }
else:
    print(f"⚠️ Unexpected number of numeric columns: {num_numeric}")
    print("\n📋 Showing first 5 rows to help identify structure:")
    print(df_combined.head().to_string())
    
    # Ask user to help
    print("\nPlease identify the columns:")
    print("The data should have columns for:")
    print("- Entrants (principal applicants)")
    print("- Derivatives (family members)")
    print("- Total (Entrants + Derivatives)")
    print("for years 2016, 2017, and 2018")
    
    # Use a flexible mapping that tries to preserve all data
    column_mapping = {}
    for i, col in enumerate(df_combined.columns[1:], 1):
        column_mapping[i] = f'Data_Column_{i}'

# Apply mapping if we have enough columns
if len(column_mapping) <= len(df_combined.columns) - 1:
    df_combined = df_combined.rename(columns=column_mapping)
    print(f"\n✅ Applied mapping for {len(column_mapping)} columns")
else:
    print("⚠️ Column mapping doesn't match data - keeping original columns")

# Keep only relevant columns
cols_to_keep = ['Country'] + list(column_mapping.values())
cols_to_keep = [c for c in cols_to_keep if c in df_combined.columns]
df_combined = df_combined[cols_to_keep]

# ============================================
# STEP 7: CONVERT TO NUMBERS
# ============================================
print("\n" + "🔢"*40)
print("🔢 STEP 6: CONVERTING TO NUMBERS")
print("🔢"*40)

def safe_number_convert(value):
    """Convert various formats to number safely"""
    if pd.isna(value):
        return 0
    
    s = str(value).strip()
    
    # Handle common non-number cases
    if s in ['', 'N/A', 'n/a', 'NA', 'na', '-', '—', '--', 'null', 'NULL']:
        return 0
    
    # Remove commas and spaces
    s = s.replace(',', '')
    s = s.replace(' ', '')
    
    # Handle ranges (take first number)
    if '-' in s and not s.startswith('-'):
        s = s.split('-')[0]
    
    try:
        # Try as integer first
        return int(float(s))
    except:
        return 0

# Convert all data columns
data_columns = [col for col in df_combined.columns if col != 'Country']
for col in data_columns:
    df_combined[col] = df_combined[col].apply(safe_number_convert)
    print(f"   ✅ Converted '{col}': sum = {df_combined[col].sum():,}")

# ============================================
# STEP 8: CALCULATE TOTALS
# ============================================
print("\n" + "🧮"*40)
print("🧮 STEP 7: CALCULATING TOTALS")
print("🧮"*40)

# Try to identify and sum entrants columns
entrants_cols = [col for col in df_combined.columns if 'Entrants' in col or 'Data_Column' in col]
if entrants_cols:
    df_combined['Total_Entrants_All_Years'] = df_combined[entrants_cols].sum(axis=1)
    print(f"✅ Total entrants across all years: {df_combined['Total_Entrants_All_Years'].sum():,}")

# Try to identify and sum derivatives columns
derivatives_cols = [col for col in df_combined.columns if 'Derivatives' in col]
if derivatives_cols:
    df_combined['Total_Derivatives_All_Years'] = df_combined[derivatives_cols].sum(axis=1)
    print(f"✅ Total derivatives across all years: {df_combined['Total_Derivatives_All_Years'].sum():,}")

# Try to identify and sum total columns
total_cols = [col for col in df_combined.columns if 'Total' in col and col != 'Total_Entrants_All_Years' and col != 'Total_Derivatives_All_Years']
if total_cols:
    df_combined['Grand_Total_All_Years'] = df_combined[total_cols].sum(axis=1)
    print(f"✅ Grand total across all years: {df_combined['Grand_Total_All_Years'].sum():,}")

# ============================================
# STEP 9: VERIFY WITH KENYA
# ============================================
print("\n" + "✅"*40)
print("✅ STEP 8: VERIFYING WITH KENYA")
print("✅"*40)

kenya_rows = df_combined[df_combined['Country'].str.contains('Kenya', case=False, na=False)]

if not kenya_rows.empty:
    kenya_data = kenya_rows.iloc[0]
    print(f"\n📊 KENYA DATA FOUND:")
    print(f"   Country: {kenya_data['Country']}")
    
    # Show all non-zero values
    has_data = False
    for col in df_combined.columns:
        if col != 'Country' and kenya_data[col] > 0:
            print(f"   {col}: {kenya_data[col]:,}")
            has_data = True
    
    if not has_data:
        print("   ⚠️ All values are ZERO for Kenya!")
        print("\n   This means the column mapping is still incorrect.")
        print("   Please run the diagnostic again and share the output.")
else:
    print("\n❌ Kenya not found in dataset!")

# ============================================
# STEP 10: DISPLAY TOP COUNTRIES
# ============================================
print("\n" + "🏆"*40)
print("🏆 STEP 9: TOP 40 COUNTRIES")
print("🏆"*40)

if 'Total_Entrants_All_Years' in df_combined.columns:
    top_countries = df_combined.sort_values('Total_Entrants_All_Years', ascending=False).head(40)
    
    print(f"\n{'Rank':<6} {'Country':<30} {'Total Applicants':>20}")
    print("-"*60)
    
    for idx, (_, row) in enumerate(top_countries.iterrows(), 1):
        print(f"{idx:<6} {row['Country'][:30]:<30} {row['Total_Entrants_All_Years']:>20,}")
else:
    print("\n⚠️ Cannot sort - total column not found")
    print("Showing first 20 rows instead:")
    print(df_combined.head(20).to_string())

# ============================================
# STEP 11: EXPORT TO EXCEL
# ============================================
print("\n" + "💾"*40)
print("💾 STEP 10: EXPORTING TO EXCEL")
print("💾"*40)

output_file = 'DV_Lottery_Extracted_Data.xlsx'
try:
    with pd.ExcelWriter(output_file) as writer:
        df_combined.to_excel(writer, sheet_name='All_Data', index=False)
        
        if 'Total_Entrants_All_Years' in df_combined.columns:
            top_countries.to_excel(writer, sheet_name='Top_20', index=False)
        
        # Summary sheet
        summary = pd.DataFrame({
            'Metric': ['Total Rows', 'Total Columns', 'File Name'],
            'Value': [len(df_combined), len(df_combined.columns), os.path.basename(pdf_path)]
        })
        summary.to_excel(writer, sheet_name='Summary', index=False)
    
    print(f"✅ Data exported to: {output_file}")
except Exception as e:
    print(f"❌ Error exporting to Excel: {e}")

# ============================================
# STEP 12: INTERACTIVE SEARCH
# ============================================
print("\n" + "🔍"*40)
print("🔍 STEP 11: INTERACTIVE SEARCH")
print("🔍 Type 'quit' to exit")
print("="*60)

while True:
    search = input("\n🌍 Enter country name: ").strip()
    
    if search.lower() == 'quit':
        print("👋 Goodbye!")
        break
    
    if not search:
        continue
    
    # Search for country
    matches = df_combined[df_combined['Country'].str.contains(search, case=False, na=False)]
    
    if len(matches) == 0:
        print(f"❌ No country matching '{search}' found")
        continue
    
    for _, row in matches.iterrows():
        print(f"\n{'='*60}")
        print(f"📊 {row['Country']}")
        print(f"{'='*60}")
        
        # Show all data columns with values
        for col in df_combined.columns:
            if col != 'Country' and row[col] > 0:
                print(f"   {col}: {row[col]:,}")
        
        # Show rank if total column exists
        if 'Total_Entrants_All_Years' in df_combined.columns and row['Total_Entrants_All_Years'] > 0:
            rank = (df_combined['Total_Entrants_All_Years'] > row['Total_Entrants_All_Years']).sum() + 1
            print(f"\n   Global Rank: #{rank} out of {len(df_combined)}")

print("\n" + "="*80)
print("✅ PROGRAM COMPLETED")
print("="*80)
