# API Testing Prompt

## Purpose and Background

This prompt is designed to help AI assistants execute comprehensive API tests for the eaststream-service project. The tests verify that the FastAPI endpoints work correctly, return accurate data, handle edge cases properly, and perform within acceptable time limits.

The test suite covers five categories:
1. **Basic functionality** — Endpoints return data, filters work, pagination works
2. **Data correctness** — Known data points (e.g., TSMC = 2330, 台積電, 半導體) are verified
3. **Boundary conditions** — Edge cases like offset exceeding total, empty results, whitespace handling
4. **Data consistency** — API responses match direct MongoDB queries
5. **Performance** — Response times are within acceptable limits

## Communication Language

When communicating with the user, always use Traditional Chinese (繁體中文). Technical output (test results, error messages) should be shown as-is.

## Execution Flow

When the user asks you to run API tests for eaststream-service, follow this process.

### Step 1: Navigate and Activate Environment

```bash
cd /Users/chouwilliam/Medina/eaststream-service
source venv/bin/activate
```

### Step 2: Run All Tests

Run the complete test suite with verbose output:

```bash
python -m pytest tests/ -v
```

This will run both sync tests and API tests. Expected total: 56+ tests.

### Step 3: Analyze Results

If all tests pass, report success with a summary like:

```
✅ 所有測試通過！

測試摘要：
- Sync 測試：15 passed
- API 測試：41 passed
- 總計：56 passed

測試類別：
- 基本功能測試 ✓
- 資料正確性測試 ✓
- 邊界條件測試 ✓
- 資料一致性測試 ✓
- 效能測試 ✓
```

If any tests fail, analyze the failure:

1. Read the error message carefully
2. Identify which category of test failed
3. Determine if it's a code bug, data issue, or test assumption problem
4. Report findings and suggest fixes

### Step 4: Run Specific Test Categories (Optional)

If the user wants to run specific categories:

```bash
# 資料正確性測試
python -m pytest tests/api/test_pdata_property.py::TestDataCorrectness -v

# 邊界條件測試
python -m pytest tests/api/test_pdata_property.py::TestBoundaryConditions -v

# 資料一致性測試
python -m pytest tests/api/test_pdata_property.py::TestDataConsistency -v

# 效能測試
python -m pytest tests/api/test_pdata_property.py::TestPerformance -v

# 基本功能測試
python -m pytest tests/api/test_pdata_property.py::TestPropertyEndpoint -v
```

### Step 5: Performance Analysis (Optional)

If the user requests performance details, run tests with timing:

```bash
python -m pytest tests/api/test_pdata_property.py::TestPerformance -v --tb=short
```

Current performance expectations:
- Single document query: < 2 seconds
- List query (1000 items): < 5 seconds
- Filtered query: < 3 seconds
- Schema endpoint: < 1 second

## Test Categories Explained

### TestDataCorrectness

Verifies known data points:
- 2330 (台積電) should have 名稱(中文)="台積電", 市場別="TSE", TSE產業名="半導體"
- 2317 (鴻海) should have 名稱(中文)="鴻海", 市場別="TSE"
- 2454 (聯發科) should have 名稱(中文)="聯發科", TSE產業名="半導體"
- 半導體產業應有 > 100 家公司
- TSE 市場應有 > 500 檔股票

### TestBoundaryConditions

Tests edge cases:
- limit=1 (minimum)
- offset exceeds total (should return empty list)
- Non-existent filter values (should return empty list)
- Whitespace in parameters (should be trimmed)
- Empty fields parameter

### TestDataConsistency

Compares API with direct MongoDB:
- Single document comparison (fields and values match)
- Total count comparison
- Filtered count comparison
- Field count comparison
- NaN values properly converted to null

### TestPerformance

Measures response times:
- All queries should complete within specified time limits
- Failures indicate potential performance issues or network problems

## Common Issues and Solutions

### Issue: NaN serialization error
**Symptom:** `ValueError: Out of range float values are not JSON compliant: nan`
**Cause:** MongoDB data contains NaN values
**Solution:** The `clean_nan_values()` function in `src/api/schemas.py` handles this

### Issue: Datetime comparison failure
**Symptom:** API returns ISO string, MongoDB returns datetime object
**Cause:** JSON serialization converts datetime to string
**Solution:** Use `normalize_for_comparison()` helper in tests

### Issue: Performance test failure
**Symptom:** Query took X seconds (expected < Y seconds)
**Cause:** Network latency, MongoDB load, or inefficient query
**Solution:** Check MongoDB indexes, network connection, or retry

## Files Reference

- Test file: `/Users/chouwilliam/Medina/eaststream-service/tests/api/test_pdata_property.py`
- API code: `/Users/chouwilliam/Medina/eaststream-service/src/api/routers/pdata.py`
- Schemas: `/Users/chouwilliam/Medina/eaststream-service/src/api/schemas.py`
- Database: `/Users/chouwilliam/Medina/eaststream-service/src/database.py`
