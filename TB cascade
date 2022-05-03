import time
from datetime import datetime
from airflow.models.dag import DAG
from airflow.decorators import task
from airflow.utils.task_group import TaskGroup
from airflow.providers.microsoft.mssql.hooks.mssql import MsSqlHook
from airflow.hooks.base_hook import BaseHook
import pandas as pd
from sqlalchemy import create_engine

#extract tasks
@task()
def get_src_tables():
    hook = MsSqlHook(mssql_conn_id="sqlserver")
    sql = """ select  t.name as table_name  
     from sys.tables t where t.name in ('DimProduct','DimRRDrugsSubcategory','DimDSDrugsSubcategory') """
    df = hook.get_pandas_df(sql)
    print(df)
    tbl_dict = df.to_dict('dict')
    return tbl_dict
#
@task()
def load_src_data(tbl_dict: dict):
    conn = BaseHook.get_connection('postgres')
    engine = create_engine(f'postgresql://{conn.login}:{conn.password}@{conn.host}:{conn.port}/{conn.schema}')
    all_tbl_name = []
    start_time = time.time()
    #access the table_name element in dictionaries
    for k, v in tbl_dict['table_name'].items():
        #print(v)
        all_tbl_name.append(v)
        rows_imported = 0
        sql = f'select * FROM {v}'
        hook = MsSqlHook(mssql_conn_id="sqlserver")
        df = hook.get_pandas_df(sql)
        print(f'importing rows {rows_imported} to {rows_imported + len(df)}... for table {v} ')
        df.to_sql(f'src_{v}', engine, if_exists='replace', index=False)
        rows_imported += len(df)
        print(f'Done. {str(round(time.time() - start_time, 2))} total seconds elapsed')
    print("Data imported successful")
    return all_tbl_name

#Transformation tasks
@task()
def transform_srcProduct():
    conn = BaseHook.get_connection('postgres')
    engine = create_engine(f'postgresql://{conn.login}:{conn.password}@{conn.host}:{conn.port}/{conn.schema}')
    pdf = pd.read_sql_query('SELECT * FROM public."src_DimProduct" ', engine)
    #drop columns
    revised = pdf[['DrugKey', 'DrugAlternateKey', 'RRtbDrugs','WeightUnitMeasureCode', 'SizeUnitMeasureCode', 'EnglishProductName',
                   'StandardCost','FinishedGoodsFlag', 'Color', 'SafetyStockLevel', 'ReorderPoint','ListPrice', 'Size', 'SizeRange', 'Weight',
                   'DaysToManufacture','BacteriogicalLine', 'ClinicalLine', 'Class', 'DrugCategory', 'PulmonaryName', 'InterventionIndicator', 'StartDate','EndDate', 'Status']]
    #replace nulls
    revised['WeightUnitMeasureCode'].fillna('0', inplace=True)
    revised['RRtbDrugs'].fillna('0', inplace=True)
    revised['SizeUnitMeasureCode'].fillna('0', inplace=True)
    revised['StandardCost'].fillna('0', inplace=True)
    revised['ListPrice'].fillna('0', inplace=True)
    revised['BacteriogicalLine'].fillna('NA', inplace=True)
    revised['Class'].fillna('NA', inplace=True)
    revised['DrugCategory'].fillna('NA', inplace=True)
    revised['Size'].fillna('NA', inplace=True)
    revised['PulmonaryName'].fillna('NA', inplace=True)
    revised['InterventionIndicator'].fillna('NA', inplace=True)
    revised['ClinicalLine'].fillna('0', inplace=True)
    revised['Weight'].fillna('0', inplace=True)
    # Rename columns with rename function
    revised = revised.rename(columns={"InterventionIndicator": "Description", "EnglishProductName":"ProductName"})
    revised.to_sql(f'stg_DimProduct', engine, if_exists='replace', index=False)
    return {"table(s) processed ": "Data imported successful"}

#
@task()
def transform_srcDrugsSubcategory():
    conn = BaseHook.get_connection('postgres')
    engine = create_engine(f'postgresql://{conn.login}:{conn.password}@{conn.host}:{conn.port}/{conn.schema}')
    pdf = pd.read_sql_query('SELECT * FROM public."src_DimRRDrugsSubcategory" ', engine)
    #drop columns
    revised = pdf[['RRtbDrugs','EnglishProductSubcategoryName', 'ProductSubcategoryAlternateKey','EnglishProductSubcategoryName', 'ProductCategoryKey']]
    # Rename columns with rename function
    revised = revised.rename(columns={"EnglishProductSubcategoryName": "ProductSubcategoryName"})
    revised.to_sql(f'stg_DimRRDrugsSubcategory', engine, if_exists='replace', index=False)
    return {"table(s) processed ": "Data imported successful"}

@task()
def transform_srcProductCategory():
    conn = BaseHook.get_connection('postgres')
    engine = create_engine(f'postgresql://{conn.login}:{conn.password}@{conn.host}:{conn.port}/{conn.schema}')
    pdf = pd.read_sql_query('SELECT * FROM public."src_DimDSDrugsSubcategory" ', engine)
    #drop columns
    revised = pdf[['ProductCategoryKey', 'ProductCategoryAlternateKey','EnglishProductCategoryName']]
    # Rename columns with rename function
    revised = revised.rename(columns={"EnglishProductCategoryName": "ProductCategoryName"})
    revised.to_sql(f'stg_DimDSDrugsSubcategory', engine, if_exists='replace', index=False)
    return {"table(s) processed ": "Data imported successful"}

#load
@task()
def prdProduct_model():
    conn = BaseHook.get_connection('postgres')
    engine = create_engine(f'postgresql://{conn.login}:{conn.password}@{conn.host}:{conn.port}/{conn.schema}')
    pc = pd.read_sql_query('SELECT * FROM public."stg_DimDSDrugsSubcategory" ', engine)
    p = pd.read_sql_query('SELECT * FROM public."stg_DimProduct" ', engine)
    p['RRtbDrugs'] = p.RRtbDrugs.astype(float)
    p['RRtbDrugs'] = p.RRtbDrugs.astype(int)
    ps = pd.read_sql_query('SELECT * FROM public."stg_DimRRDrugsSubcategory" ', engine)
    #join all three
    merged = p.merge(ps, on='RRtbDrugs').merge(pc, on='ProductCategoryKey')
    merged.to_sql(f'prd_DimDSDrugsSubcategory', engine, if_exists='replace', index=False)
    return {"table(s) processed ": "Data imported successful"}


# [START how_to_task_group]
with DAG(dag_id="product_etl_dag",schedule_interval="0 9 * * *", start_date=datetime(2022, 3, 5),catchup=False,  tags=["product_model"]) as dag:

    with TaskGroup("extract_dimProudcts_load", tooltip="Extract and load source data") as extract_load_src:
        src_product_tbls = get_src_tables()
        load_dimProducts = load_src_data(src_product_tbls)
        #define order
        src_product_tbls >> load_dimProducts

    # [START howto_task_group_section_2]
    with TaskGroup("transform_src_product", tooltip="Transform and stage data") as transform_src_product:
        transform_srcProduct = transform_srcProduct()
        transform_srcProductSubcategory = transform_srcProductSubcategory()
        transform_srcProductCategory = transform_srcProductCategory()
        #define task order
        [transform_srcProduct, transform_srcProductSubcategory, transform_srcProductCategory]

    with TaskGroup("load_product_model", tooltip="Final Product model") as load_product_model:
        prd_Product_model = prdProduct_model()
        #define order
        prd_Product_model

    extract_load_src >> transform_srcProductCategory >> load_product_model
