# Capstone 2025 - MLB Relief Pitcher Rest Day Analysis

## Project Overview
Research project analyzing the relationship between rest days and performance metrics (velocity, spin rate, control) for MLB relief pitchers using Statcast pitch-level data (2021-2024).

## Data Architecture

### Data Pipeline (Sequential Order)
1. **[data_collection_capstone.ipynb](../data_collection_capstone.ipynb)**: Downloads raw Statcast data via `pybaseball`
   - Fetches full season data by year using date ranges
   - Saves each year separately as Parquet: `data/statcast_YYYY.parquet`
   - Memory management: Sets variables to `None` after saving to free memory
   
2. **[data_managing_reliever_table_capstone.ipynb](../data_managing_reliever_table_capstone.ipynb)**: Creates cleaned relief pitcher dataset
   - Filters 31 relevant columns from raw Statcast data
   - **Critical filtering logic**: Identifies starters as pitchers with 3+ starts in a season (inning 1 appearances)
   - Adds pitcher names via `pybaseball.playerid_reverse_lookup()`
   - Applies activity thresholds: 20+ games AND 30+ innings pitched per season
   - Output: `data/statcast_relievers_final.parquet`

3. **[data_managing_summary_stats_capstone.ipynb](../data_managing_summary_stats_capstone.ipynb)**: Calculates rest days and aggregates game-level statistics

### Key Data Files
- **Raw Data**: `data/statcast_YYYY.parquet` (one per year, pitch-level)
- **Processed Data**: `data/statcast_relievers_final.parquet` (relief appearances only, ~2.9M pitches)

## Python Environment

### Required Packages
```python
pybaseball  # MLB data retrieval
polars      # Primary data manipulation (preferred over pandas)
pandas      # Used for pybaseball compatibility
numpy       # Numerical operations
```

### Polars-First Approach
This project uses **Polars** as the primary data manipulation library (not pandas). Key patterns:
- Read files: `pl.read_parquet('path')`
- Filtering: `.filter(pl.col('column') == value)`
- Aggregation: `.group_by(['col1', 'col2']).agg(pl.col('metric').mean())`
- Column selection: `.select(['col1', 'col2'])`
- Renaming: `.rename({'old': 'new'})`
- Adding columns: `.with_columns(pl.lit(value).alias('new_col'))`
- Combining: `pl.concat([df1, df2, df3])`

### Pandas Usage (Limited)
Only use pandas for `pybaseball` functions that require it:
- `playerid_reverse_lookup()` returns pandas DataFrame
- Convert to Polars: `pl.from_pandas(df)`

## Notebook Execution Workflow

### Memory Management
When working with large datasets (2.9M+ rows):
1. Load one year at a time when downloading
2. Set variables to `None` after saving to free memory
3. Use `.select()` to filter columns early before combining years

### Typical Cell Execution Pattern
1. Import packages (run once per session)
2. Load data files
3. Transform/filter data
4. Verify with `.shape`, `.head()`, or `.n_unique()`
5. Save results to Parquet
6. Clear large intermediate variables if needed

## Project-Specific Patterns

### Relief Pitcher Identification
**Critical logic**: Starting pitchers are identified as the first pitcher who threw in inning 1 of each game-half. This approach:
- Excludes only specific starting appearances (not entire pitchers)
- Includes relief appearances by pitchers who occasionally start
- Creates starter set: `(game_id, top_bottom, pitcher_id)` combinations

Example:
```python
starters = (statcast_combined
    .filter(pl.col('inning') == 1)
    .group_by(['game_id', 'top_bottom'])
    .agg(pl.col('pitcher_id').first()))
```

### Column Naming Convention
Variables follow consistent renaming from raw Statcast:
- `pitcher` → `pitcher_id`
- `game_date` → `date`
- `release_speed` → `velocity`
- `release_spin_rate` → `spin_rate`
- `pitch_name` → `pitch_type`

### Data Validation Steps
After major transformations, verify:
- Row counts match expectations
- Unique pitcher/game counts by year
- No null values in key columns
- Data sorted properly: `['pitcher_id', 'date', 'game_id', 'pitch_num']`

## Performance Considerations
- Parquet files are preferred format (efficient compression, columnar storage)
- Sort data early for time-series calculations (rest days)
- Use Polars for large datasets (faster than pandas)
- Filter columns before combining years to reduce memory

## Common Tasks

### Adding New Variables
When adding calculated columns (e.g., rest days):
1. Ensure data is sorted by pitcher and date
2. Use `.with_columns()` with Polars expressions
3. Verify calculations on sample pitchers before full dataset

### Extending Date Range
To add 2025 data:
1. Add new cell in [data_collection_capstone.ipynb](../data_collection_capstone.ipynb) with appropriate dates
2. Update list of files in [data_managing_reliever_table_capstone.ipynb](../data_managing_reliever_table_capstone.ipynb)
3. Verify column consistency across years

### Pitcher Lookup
Use `pybaseball.playerid_reverse_lookup(pitcher_ids, key_type='mlbam')` to get names from IDs.
