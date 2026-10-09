
"""
Interactive Data Explorer
Run with: streamlit run interactive_data_explorer.py
"""

from io import StringIO

import pandas as pd
import plotly.express as px
import streamlit as st


st.set_page_config(
    page_title="Interactive Data Explorer",
    page_icon="🔎",
    layout="wide",
)

st.title("🔎 Interactive Data Explorer")
st.write("Upload, filter, visualize, and export your data.")

SAMPLE_CSV = """date,department,region,sales,orders,customer_rating
2026-01-03,Electronics,East,12500,42,4.5
2026-01-08,Home,West,8200,31,4.1
2026-01-14,Books,North,3400,58,4.7
2026-02-02,Electronics,West,15100,49,4.4
2026-02-11,Home,East,7600,28,3.9
2026-02-19,Books,South,4100,64,4.8
2026-03-04,Electronics,North,13800,45,4.3
2026-03-12,Home,South,9100,35,4.2
2026-03-21,Books,East,3900,61,4.6
2026-04-05,Electronics,South,17200,54,4.6
2026-04-13,Home,North,8700,33,4.0
2026-04-25,Books,West,4500,67,4.9
2026-05-02,Electronics,East,16400,51,4.5
2026-05-15,Home,West,9800,37,4.3
2026-05-23,Books,South,4800,70,4.8
2026-06-06,Electronics,West,18100,57,4.7
2026-06-17,Home,East,10300,39,4.2
2026-06-26,Books,North,5200,73,4.9
"""


def read_csv(file):
    """Read CSV files with UTF-8 or Japanese Windows encoding."""
    try:
        return pd.read_csv(file)
    except UnicodeDecodeError:
        file.seek(0)
        return pd.read_csv(file, encoding="cp932")


def detect_dates(data):
    """Automatically detect columns with date-like names."""
    data = data.copy()

    for column in data.columns:
        name = str(column).lower()

        if any(word in name for word in
               ("date", "time", "日付", "日時")):
            parsed = pd.to_datetime(
                data[column], errors="coerce"
            )

            if parsed.notna().mean() >= 0.7:
                data[column] = parsed

    return data


# -----------------------------
# Sidebar: data source
# -----------------------------

st.sidebar.header("1. Choose your data")

uploaded_file = st.sidebar.file_uploader(
    "Upload a CSV file",
    type=["csv"],
)

if uploaded_file is not None:
    try:
        original_data = read_csv(uploaded_file)
    except Exception as error:
        st.error(f"Could not read CSV: {error}")
        st.stop()
else:
    st.sidebar.info("Using the sample sales dataset.")
    original_data = pd.read_csv(StringIO(SAMPLE_CSV))

if original_data.empty:
    st.warning("The dataset contains no rows.")
    st.stop()

data = detect_dates(original_data)

date_columns = [
    c for c in data.columns
    if pd.api.types.is_datetime64_any_dtype(data[c])
]

numeric_columns = [
    c for c in data.columns
    if pd.api.types.is_numeric_dtype(data[c])
]

category_columns = [
    c for c in data.columns
    if c not in date_columns
    and c not in numeric_columns
    and data[c].nunique(dropna=True) <= 100
]

# -----------------------------
# Sidebar: interactive filters
# -----------------------------

st.sidebar.header("2. Filter your data")

filtered = data.copy()

for column in date_columns:
    dates = filtered[column].dropna()

    if dates.empty:
        continue

    start, end = st.sidebar.date_input(
        f"Date range: {column}",
        value=(dates.min().date(), dates.max().date()),
        key=f"date_{column}",
    )

    start = pd.Timestamp(start)
    end = (
        pd.Timestamp(end)
        + pd.Timedelta(days=1)
        - pd.Timedelta(microseconds=1)
    )

    filtered = filtered[
        filtered[column].isna()
        | filtered[column].between(start, end)
    ]

for column in numeric_columns:
    values = pd.to_numeric(
        filtered[column], errors="coerce"
    ).dropna()

    if values.empty or values.min() == values.max():
        continue

    low, high = float(values.min()), float(values.max())

    selected_range = st.sidebar.slider(
        f"Range: {column}",
        min_value=low,
        max_value=high,
        value=(low, high),
        key=f"number_{column}",
    )

    filtered = filtered[
        filtered[column].isna()
        | filtered[column].between(
            selected_range[0], selected_range[1]
        )
    ]

for column in category_columns:
    choices = sorted(
        filtered[column].dropna().astype(str).unique()
    )

    selected = st.sidebar.multiselect(
        f"Select {column}",
        options=choices,
        default=choices,
        key=f"category_{column}",
    )

    filtered = filtered[
        filtered[column].isna()
        | filtered[column].astype(str).isin(selected)
    ]

# -----------------------------
# Overview metrics
# -----------------------------

st.subheader("Dataset overview")

col1, col2, col3, col4 = st.columns(4)

col1.metric("Rows", f"{len(filtered):,}")
col2.metric("Columns", f"{len(filtered.columns):,}")
col3.metric(
    "Missing cells",
    f"{int(filtered.isna().sum().sum()):,}",
)
col4.metric(
    "Duplicate rows",
    f"{int(filtered.duplicated().sum()):,}",
)

# -----------------------------
# Main tabs
# -----------------------------

tab_data, tab_stats, tab_charts, tab_quality = st.tabs([
    "📋 Data",
    "📊 Statistics",
    "📈 Charts",
    "🧹 Data quality",
])

with tab_data:
    st.subheader("Filtered records")

    st.dataframe(
        filtered,
        use_container_width=True,
        hide_index=True,
    )

    csv_data = filtered.to_csv(
        index=False
    ).encode("utf-8-sig")

    st.download_button(
        "Download filtered CSV",
        data=csv_data,
        file_name="filtered_data.csv",
        mime="text/csv",
    )

with tab_stats:
    st.subheader("Descriptive statistics")

    if numeric_columns:
        st.dataframe(
            filtered[numeric_columns].describe().T,
            use_container_width=True,
        )
    else:
        st.info("No numeric columns found.")

    profile = pd.DataFrame({
        "Column": filtered.columns,
        "Data type": [
            str(filtered[c].dtype) for c in filtered.columns
        ],
        "Non-null": [
            int(filtered[c].notna().sum())
            for c in filtered.columns
        ],
        "Missing": [
            int(filtered[c].isna().sum())
            for c in filtered.columns
        ],
        "Unique": [
            int(filtered[c].nunique(dropna=True))
            for c in filtered.columns
        ],
    })

    st.subheader("Column profile")
    st.dataframe(profile, use_container_width=True)

with tab_charts:
    st.subheader("Interactive chart builder")

    chart_type = st.selectbox(
        "Choose chart type",
        [
            "Bar chart",
            "Line chart",
            "Scatter plot",
            "Histogram",
            "Box plot",
            "Correlation heatmap",
        ],
    )

    if chart_type == "Correlation heatmap":
        if len(numeric_columns) >= 2:
            fig = px.imshow(
                filtered[numeric_columns].corr(),
                text_auto=".2f",
                title="Correlation between numeric columns",
                aspect="auto",
            )
            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("Select a dataset with at least two numeric columns.")

    elif chart_type == "Histogram":
        if numeric_columns:
            x = st.selectbox("Numeric column", numeric_columns)
            bins = st.slider("Number of bins", 5, 80, 25)

            fig = px.histogram(
                filtered,
                x=x,
                nbins=bins,
                title=f"Distribution of {x}",
            )
            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("No numeric columns available.")

    elif chart_type == "Box plot":
        if numeric_columns:
            y = st.selectbox("Numeric column", numeric_columns)
            groups = ["None"] + category_columns
            group = st.selectbox("Group by", groups)

            fig = px.box(
                filtered,
                x=None if group == "None" else group,
                y=y,
                title=f"Distribution of {y}",
            )
            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("No numeric columns available.")

    elif chart_type in ("Bar chart", "Line chart"):
        x_options = list(dict.fromkeys(
            date_columns + category_columns + numeric_columns
        ))

        if x_options and numeric_columns:
            x = st.selectbox("X-axis", x_options)
            y = st.selectbox("Y-axis", numeric_columns)
            aggregation = st.selectbox(
                "Aggregation",
                ["Sum", "Mean", "Median", "Count"],
            )

            color_options = ["None"] + category_columns
            color = st.selectbox("Group/color", color_options)

            if x in date_columns or x in category_columns:
                if aggregation == "Sum":
                    plot_data = filtered.groupby(
                        x, dropna=False, as_index=False
                    )[y].sum()
                elif aggregation == "Mean":
                    plot_data = filtered.groupby(
                        x, dropna=False, as_index=False
                    )[y].mean()
                elif aggregation == "Median":
                    plot_data = filtered.groupby(
                        x, dropna=False, as_index=False
                    )[y].median()
                else:
                    plot_data = filtered.groupby(
                        x, dropna=False, as_index=False
                    )[y].count()

                if color != "None" and color != x:
                    grouped = filtered.groupby(
                        [x, color], dropna=False, as_index=False
                    )[y]

                    if aggregation == "Sum":
                        plot_data = grouped.sum()
                    elif aggregation == "Mean":
                        plot_data = grouped.mean()
                    elif aggregation == "Median":
                        plot_data = grouped.median()
                    else:
                        plot_data = grouped.count()

                    color_arg = color
                else:
                    color_arg = None
            else:
                plot_data = filtered
                color_arg = None if color == "None" else color

            if chart_type == "Line chart":
                fig = px.line(
                    plot_data,
                    x=x,
                    y=y,
                    color=color_arg,
                    markers=True,
                )
            else:
                fig = px.bar(
                    plot_data,
                    x=x,
                    y=y,
                    color=color_arg,
                )

            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("This chart requires a numeric column.")

    elif chart_type == "Scatter plot":
        if len(numeric_columns) >= 2:
            x = st.selectbox("X-axis", numeric_columns)
            y_options = [c for c in numeric_columns if c != x]
            y = st.selectbox("Y-axis", y_options)

            color_options = ["None"] + category_columns
            color = st.selectbox("Color by", color_options)

            fig = px.scatter(
                filtered,
                x=x,
                y=y,
                color=None if color == "None" else color,
                hover_data=list(
                    filtered.columns[:min(5, len(filtered.columns))]
                ),
                title=f"{y} vs. {x}",
            )

            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("Scatter plots require two numeric columns.")

with tab_quality:
    st.subheader("Data quality report")

    quality = pd.DataFrame({
        "Column": data.columns,
        "Missing values": [
            int(data[c].isna().sum()) for c in data.columns
        ],
        "Missing (%)": [
            round(data[c].isna().mean() * 100, 2)
            for c in data.columns
        ],
        "Unique values": [
            int(data[c].nunique(dropna=True))
            for c in data.columns
        ],
        "Data type": [
            str(data[c].dtype) for c in data.columns
        ],
    }).sort_values("Missing (%)", ascending=False)

    st.dataframe(quality, use_container_width=True)

    st.write(
        f"Duplicate rows: {int(data.duplicated().sum()):,}"
    )

    if st.button("Remove duplicate rows and prepare download"):
        deduplicated = filtered.drop_duplicates()

        st.success(
            f"{len(filtered) - len(deduplicated)} duplicate rows removed."
        )

        st.dataframe(deduplicated, use_container_width=True)

        st.download_button(
            "Download cleaned CSV",
            data=deduplicated.to_csv(
                index=False
            ).encode("utf-8-sig"),
            file_name="cleaned_data.csv",
            mime="text/csv",
        )

st.divider()
st.caption("Built with Python, Streamlit, pandas, and Plotly.")
