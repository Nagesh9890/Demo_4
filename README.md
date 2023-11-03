from pyspark.sql import SparkSession
from pyspark.sql.functions import udf, col, when, lit
from pyspark.sql.types import StringType, StructType, StructField, ArrayType
import pandas as pd
import ast

# Initialize a Spark session
spark = SparkSession.builder.appName("Classification").getOrCreate()

# Read the CSV file using pandas
keywords_df = pd.read_csv("classification_keywords.csv")

# Convert the pandas DataFrame to a dictionary for keyword lookup
keywords_dict = {}
for index, row in keywords_df.iterrows():
    keywords_str = str(row['KEYWORDS']).strip()
    try:
        if keywords_str.startswith("[") and keywords_str.endswith("]"):
            keywords_list = ast.literal_eval(keywords_str)
        else:
            keywords_list = [keyword.strip() for keyword in keywords_str.split(",")]
        for keyword in keywords_list:
            keywords_dict[keyword.lower()] = (row['CATEGORY_LEVEL1'], row['CATEGORY_LEVEL2'])
    except Exception as e:
        print(f"Skipping row at index {index}: {e}")

# Function to set category levels
def set_category_levels(base_txn_text, benef_name):
    base_txn_text = base_txn_text.lower() if base_txn_text else ''
    benef_name = benef_name.lower() if benef_name else ''
    
    for keyword, (cat_level1, cat_level2) in keywords_dict.items():
        if keyword in base_txn_text or keyword in benef_name:
            return [cat_level1, cat_level2]
    
    return ['OTHER TRANSFER', 'OTHER']

# Define the schema for the UDF return type
schema = StructType([
    StructField("category_level1", StringType(), True),
    StructField("category_level2", StringType(), True)
])

# Register the UDF
category_udf = udf(set_category_levels, schema)

# Define the UDF for classification
def classify_transaction(benef_ifsc, benef_account_no, source, benef_name):
    corporate_keywords = [
        "pvt ltd", "innovation", "tata", "steel", "industry", "llp",
        "corporation", "institutional", "tech", "automobiles", "services",
        "telecommunication", "travels"
    ]
    
    def contains_corporate_keyword(name):
        return any(keyword in name.lower() for keyword in corporate_keywords)
    
    is_corporate = contains_corporate_keyword(benef_name)
    
    if benef_ifsc and benef_ifsc.startswith("YESB"):
        if source == 'current':
            return 'YBL_Corp'
        elif source == 'saving':
            return 'YBL_Ind'
        else:  # Treat source being None or empty string as needing benef_name check
            return 'YBL_Corp' if is_corporate else 'YBL_Ind'
    else:
        return 'non_ybl_cor' if is_corporate else 'non_ybl_ind'

classify_transaction_udf = udf(classify_transaction, StringType())

# Sample data
data = [
    ("John Doe", "Credit Card Payment", "XYZ Corp", "YESB0000001", "123456789012345", "current"),
    ("Jane Smith", "Shopping at Amazon", "Jane Smith", "HDFC0000001", "1234567890123", "saving"),
    ("Alice Johnson", "self expense", "Alice Johnson", "YESB0000001", "123456789012345", "saving"),
    ("Bob Brown", "rtgs transfer", "ACME Corp", "YESB0000001", "123456789012345", "current"),
    ("Charlie Davis", "taxi rent uber", "Uber Technologies", "ICIC0000001", "1234567890123", "saving"),
    ("Eve Taylor", "health insurance", "Global Insure", "HDFC0000001", "123456789012345", "")
]
columns = ["remitter_name", "base_txn_text", "benef_name", "benef_ifsc", "benef_account_no", "source"]

# Create the DataFrame
df = spark.createDataFrame(data, columns)

# Perform the classification
df = df.withColumn('cor_ind_benf', classify_transaction_udf(col('benef_ifsc'), col('benef_account_no'), col('source'), col('benef_name')))

# Use the UDF to get the category levels and split the results into two new columns
category_cols = category_udf(col('base_txn_text'), col('benef_name'))
df = df.withColumn('category_levels', category_cols)

df = df.withColumn('category_level1', df['category_levels'].getItem(0)) \
       .withColumn('category_level2', df['category_levels'].getItem(1)) \
       .drop('category_levels')

# Override category_level2 if remitter_name equals benef_name
df = df.withColumn('category_level2', when(col('remitter_name').lower() == col('benef_name').lower(), lit('Personal Transfer'))
                                    .otherwise(col('category_level2')))

# Show the DataFrame
df.show(truncate=False)

# Stop the Spark session
spark.stop()

