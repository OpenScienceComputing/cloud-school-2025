## 🌊 Cloud-Native Solutions for Met/Ocean Forecast Data

A fundamental challenge in **meteorological and oceanographic (met/ocean) forecasting** is the efficient distribution of forecast model results. Standard forecast models typically run daily (e.g., a 3-day forecast run every day), creating a collection of files with **overlapping time coordinates**. End-users, however, almost always require a **continuous time series** (e.g., a "best time series") to simplify analysis and comparison with observational data.

Historically, data providers addressed this by:

  * Creating new, consolidated "best time series" datasets by cutting and merging segments from the individual forecast runs.
  * Utilizing data services like **THREDDS** to create virtual Forecast Model Run Collection (FMRC) datasets, offering views like "best time series" or "constant forecast."

-----

### The Modern Cloud-Native Approach

For **cloud-native workflows**, a more scalable, flexible, and robust approach is now available. This modern strategy leverages specialized tools to construct virtual data views dynamically, avoiding expensive data copying and reformatting:

1.  **Virtualizarr** is used to create virtual references (metadata) for the model output files.
2.  These references are indexed into an **Icechunk repository**.
3.  **Rolodex** then uses **Xarray's advanced indexing capabilities** to construct the required views (like the "best time series") on the fly, providing immediate access to continuous data streams.

This exact pipeline has recently been implemented to support the **CoastPredict/GlobalCoast/Protocoast** project.

-----

## 🌍 The GlobalCoast and ProtoCoast Initiatives

**CoastPredict** is a Programme endorsed under the UN Decade of Ocean Science for Sustainable Development, focused on revolutionizing Global Coastal Ocean observation and forecasting.

**GlobalCoast** is a European-led endorsed action under the UN Decade of Ocean Science for Sustainable Development. It provides the framework for establishing a seamless, integrated, and sustained **global coastal ocean observing and forecasting system**. This system aims to improve the understanding of coastal processes, enhance the prediction of coastal hazards, and deliver essential information for the sustainable management of coastal resources, addressing critical issues like sea-level rise and marine pollution.

### ProtoCoast: Enabling Cloud-Native Workflows

To standardize computation and accelerate progress, **ProtoCoast** is a key GlobalCoast initiative focused on enabling **cloud-native workflows** for model execution, data accessibility (both observational and model output), and the creation of shared research environments.

ProtoCoast utilizes the **EGI Cloud Infrastructure**. Initial workflow testing has been conducted on the **Pangeo@EOSC JupyterHub** funded by European Union Horizon 2020 projects (EGI-ACE, and C-SCALE).  These projects worked with the Pangeo Europe Community and EGI (E-infrastructure Grid) to deploy the platform—a DaskHub composed of Dask Gateway and JupyterHub—on the EGI Cloud Compute infrastructure within the European Open Science Cloud (EOSC). The infrastructure provision also received an in-kind contribution from the e-INFRA CZ project. 

-----

## ⛵️ Pilot Site Example: Gulf of Taranto

ProtoCoast features several pilot sites producing both forecast model output and near-real-time sensor data. The **Gulf of Taranto** pilot site provides an excellent example. It features:

  * **Near-Real Time Data:** Telemetered water level sensor data.
  * **Model Output:** The **SHYFEM coastal ocean model** runs daily, producing a 6-day forecast stored in a **NetCDF3 file**.

-----

### Step 1: Rechunking and Cloud Ingestion

The initial step in the cloud-native data pipeline is preparing the model output for efficient cloud access.

  * The two NetCDF3 files produced by SHYFEM are **reformatted and rechunked** on the High-Performance Computing (HPC) system where the model runs.
  * The **NCO (NetCDF Operators)** tool is used for this process, converting the data to **NetCDF4 (optimized for cloud I/O)**. This choice retains the NetCDF format to support existing legacy applications while gaining cloud-optimized features.

The rechunking command specifies the new chunk sizes for better performance:

```bash
ncks -4 -L 5 -O -d time,0,143 --cnk_dmn=time,72 --cnk_dmn=node,16000 --cnk_dmn=level,1 --cnk_plc=all taranto_nos.nc taranto_nos_20251205_nc4.nc
```

This configuration results in approximately **4MB chunks** for both 2D (like elevation) and 3D (like temperature) variables. These rechunked, cloud-ready files are then pushed to an **S3-compatible (MinIO) bucket** on the EGI Cloud infrastructure.

-----

### Step 2: Virtualization and Indexing with Virtualizarr, Icechunk, and Rolodex

Once the files are in the cloud, a Python script running on the HPC system executes the core virtualization process. This involves:

1.  **Creating Virtual References:** Virtualizarr generates references to the data.
2.  **FMRC Convention Formatting:** The single `time` coordinate (hourly steps) in the raw NetCDF files is converted to the Forecast Model Run Collection (FMRC) convention required by Rolodex, resulting in two new coordinate variables: `valid_time` (the analysis time) and `step` (the forecast period offset).
3.  **Appending to Icechunk:** The references are then appended to a virtual Icechunk repository along the new `valid_time` dimension.

The code to transform and combine the datasets looks like this:

```python
ds_list = [
    open_virtual_dataset(
        url=url,
        parser=parser,
        registry=registry,
        loadable_variables=["time"],
    )
    for url in flist[-1:]
]

def fix_ds(ds):
    ds = ds.rename_vars(time='valid_time')
    ds = ds.rename_dims(time='step')
    step = (ds.valid_time - ds.valid_time[0]).assign_attrs({"standard_name": "forecast_period"})
    time = ds.valid_time[0].assign_attrs({"standard_name": "forecast_reference_time"})
    ds = ds.assign_coords(step=step, time=time)
    ds = ds.drop_indexes("valid_time")
    ds = ds.drop_vars('valid_time')
    ds = ds.set_coords(['latitude', 'longitude', 'element_index', 'topology', 'total_depth'])
    return ds

ds_list = [fix_ds(ds) for ds in ds_list]

combined_nos = xr.concat(
    ds_list,
    dim="time",
    coords="minimal",
    compat="override",
    combine_attrs="override",
)
```

The resulting 2D and 3D model outputs are merged and appended to the Icechunk repository:

```python
ds = xr.merge([combined_nos, combined_ous], compat='override')
ds.virtualize.to_icechunk(append_session.store, append_dim="time")
```

A full notebook version of this script is [here](https://www.google.com/search?q=https://github.com/OpenScienceComputing/cloud-school-2025/blob/main/taranto-icechunk-append.ipynb).

The resulting virtual dataset in Xarray, now structured for FMRC indexing, appears as:
<img width="700" alt="Screenshot 2025-12-02 092413" src="https://github.com/user-attachments/assets/49c0b169-d86f-4764-bd09-5768bb370605" />


### Dynamic Views with Rolodex

Finally, **Rolodex** is used to extract a **"best time series"** view dynamically. We select a constant forecast offset (e.g., 2 hours after the analysis time) for the continuous series:

```python
import rolodex.forecast
from rolodex.forecast import (
    BestEstimate,
    ForecastIndex,
)

ds.coords["valid_time"] = rolodex.forecast.create_lazy_valid_time_variable(
    reference_time=ds.time, period=ds.step
)

newds = ds.drop_indexes(["time", "step"]).set_xindex(
    ["time", "step", "valid_time"], ForecastIndex)

ds_best = newds.sel(valid_time=BestEstimate(offset=2))  # start at forecast hour 2 instead of 0
```
This produces a dynamically indexed dataset, effectively a continuous time series:
<img width="700" alt="Screenshot 2025-12-02 092738" src="https://github.com/user-attachments/assets/18540858-f761-46af-b177-cba9dd188085" />


This cloud-native pipeline allows for rapid extraction and plotting of time series data at a specific location, taking less than 3 seconds for the operation.
<img width="700"  alt="Screenshot 2025-12-02 092836" src="https://github.com/user-attachments/assets/2ba53f37-a1f7-402e-bc66-d62e92adbfc7" />

Full notebook [here](https://www.google.com/search?q=https://github.com/OpenScienceComputing/cloud-school-2025/blob/main/taranto-icechunk-FMRC.ipynb).

-----

## 💡 Conclusion

The implementation of this pipeline for the **GlobalCoast/Protocoast** project successfully demonstrates a highly scalable and resilient approach to distributing complex met/ocean forecast data. By shifting from traditional data copying/merging to a **cloud-native virtualization strategy** leveraging **Virtualizarr, Icechunk, and Rolodex**, we have enabled **on-the-fly FMRC views** like the "best time series." This modern architecture ensures that end-users—whether running analysis on the **Pangeo@EOSC JupyterHub** or supporting critical coastal management decisions—have immediate, performant access to the continuous, authoritative data they require, all while streamlining the data provider's workflow and maximizing the utility of the EGI Cloud Infrastructure.
