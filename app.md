import streamlit as st
import pandas as pd
import geopandas
import plotly.express as px
import numpy as np
from shapely.geometry import Point
import os

# 서울시 25개 자치구 리스트
seoul_districts_25 = [
    '강남구', '강동구', '강북구', '강서구', '관악구', '광진구', '구로구', '금천구', '노원구',
    '도봉구', '동대문구', '동작구', '마포구', '서대문구', '서초구', '성동구', '성북구', '송파구',
    '양천구', '영등포구', '용산구', '은평구', '종로구', '중구', '중랑구'
]

@st.cache_data
def load_and_process_all_data():
    dashboard_data_df = pd.DataFrame()
    geojson_data = {}
    gdf_seoul_for_map = geopandas.GeoDataFrame()
    seoul_district_areas_df = pd.DataFrame()

    try:
        # 경로 설정 (./data/)
        geojso_file_path = './data/BND_SIGUNGU_PG.shp'
        gdf_seoul = geopandas.read_file(geojso_file_path, encoding='cp949')
        gdf_seoul_renamed = gdf_seoul.rename(columns={'SIGUNGU_NM': '자치구_코드_명'})
        gdf_seoul_renamed = gdf_seoul_renamed[gdf_seoul_renamed.geometry.is_valid].copy()

        gdf_seoul_filtered = gdf_seoul_renamed[
            gdf_seoul_renamed['자치구_코드_명'].isin(seoul_districts_25)
        ].copy()
        gdf_seoul_final_for_merge = gdf_seoul_filtered.dissolve(by='자치구_코드_명', aggfunc='first').reset_index()

        reprojected_gdf_25 = gdf_seoul_final_for_merge.to_crs(epsg=5179)
        reprojected_gdf_25['면적(km²)'] = reprojected_gdf_25.geometry.area / 1_000_000
        seoul_district_areas_df = reprojected_gdf_25[['자치구_코드_명', '면적(km²)']].copy()

        gdf_seoul_for_map = gdf_seoul_final_for_merge.to_crs('EPSG:4326')
        geojson_data = gdf_seoul_for_map.__geo_interface__
    except Exception as e:
        st.error(f"지리 데이터 로드/처리 중 오류 발생: {e}")
        return pd.DataFrame(), {}, geopandas.GeoDataFrame()

    # 인구 데이터
    file_path_population = './data/서울시 상권분석서비스(상주인구-자치구).csv'
    try:
        df_population = pd.read_csv(file_path_population, encoding='cp949')
        average_population_by_district = df_population.groupby('자치구_코드_명')['총_상주인구_수'].mean().reset_index()
        average_population_by_district['순위'] = average_population_by_district['총_상주인구_수'].rank(ascending=False, method='min').astype(int)

        seoul_population_df = average_population_by_district[
            average_population_by_district['자치구_코드_명'].isin(seoul_districts_25)
        ].copy()
        merged_population_density_df = pd.merge(
            seoul_population_df,
            seoul_district_areas_df,
            on='자치구_코드_명',
            how='inner'
        )
        merged_population_density_df['인구_밀도(명/km²)'] = (
            merged_population_density_df['총_상주인구_수'] / merged_population_density_df['면적(km²)' ]
        )
    except Exception as e:
        st.warning(f"인구 데이터 로드/처리 중 오류 발생: {e}")
        merged_population_density_df = pd.DataFrame()

    # 정류장 데이터
    file_path_station_info = './data/GGD_StationInfo_M.xlsx'
    seoul_bus_stops_df = pd.DataFrame()
    seoul_subway_stations_df = pd.DataFrame()
    merged_bus_density_df = pd.DataFrame()
    merged_subway_density_df = pd.DataFrame()
    seoul_transport_lacking_df = pd.DataFrame()
    try:
        df_station_info_raw = pd.read_excel(file_path_station_info)
        df_station_info_raw['X'] = pd.to_numeric(df_station_info_raw['X'], errors='coerce')
        df_station_info_raw['Y'] = pd.to_numeric(df_station_info_raw['Y'], errors='coerce')
        df_station_info_raw.dropna(subset=['X', 'Y'], inplace=True)

        geometry_stations = [Point(xy) for xy in zip(df_station_info_raw['X'], df_station_info_raw['Y'])]
        gdf_all_stations = geopandas.GeoDataFrame(df_station_info_raw, geometry=geometry_stations, crs='EPSG:4326')

        gdf_seoul_final_for_merge_reprojected_4326 = gdf_seoul_final_for_merge.to_crs('EPSG:4326')

        all_stations_with_districts = geopandas.sjoin(
            gdf_all_stations,
            gdf_seoul_final_for_merge_reprojected_4326[['자치구_코드_명', 'geometry']],
            how='inner',
            predicate='within'
        )
        seoul_bus_stops_df = all_stations_with_districts.groupby('자치구_코드_명').size().reset_index(name='버스정류장_수')
        seoul_subway_stations_df = all_stations_with_districts.groupby('자치구_코드_명').size().reset_index(name='지하철역_수')

        merged_bus_density_df = pd.merge(
            seoul_bus_stops_df,
            seoul_district_areas_df,
            on='자치구_코드_명',
            how='inner'
        )
        merged_bus_density_df['버스정류장_밀도(개/km²)'] = merged_bus_density_df['버스정류장_수'] / merged_bus_density_df['면적(km²)' ]

        merged_subway_density_df = pd.merge(
            seoul_subway_stations_df,
            seoul_district_areas_df,
            on='자치구_코드_명',
            how='inner'
        )
        merged_subway_density_df['지하철역_밀도(개/km²)'] = merged_subway_density_df['지하철역_수'] / merged_subway_density_df['면적(km²)' ]

        seoul_public_transport_counts_df = pd.merge(
            seoul_bus_stops_df,
            seoul_subway_stations_df,
            on='자치구_코드_명',
            how='outer'
        )
        seoul_public_transport_counts_df['버스정류장_수'] = seoul_public_transport_counts_df['버스정류장_수'].fillna(0).astype(int)
        seoul_public_transport_counts_df['지하철역_수'] = seoul_public_transport_counts_df['지하철역_수'].fillna(0).astype(int)

        seoul_public_transport_counts_df['정류장_부족_순위'] = \
            seoul_public_transport_counts_df['버스정류장_수'].rank(ascending=True, method='min').astype(int)
        seoul_public_transport_counts_df['지하철_부족_순위'] = \
            seoul_public_transport_counts_df['지하철역_수'].rank(ascending=True, method='min').astype(int)
        seoul_public_transport_counts_df['종합_교통_부족_순위'] = \
            seoul_public_transport_counts_df['정류장_부족_순위'] + seoul_public_transport_counts_df['지하철_부족_순위']

        seoul_transport_lacking_df = seoul_public_transport_counts_df[['자치구_코드_명', '정류장_부족_순위', '지하철_부족_순위', '종합_교통_부족_순위']].copy()

    except Exception as e:
        st.warning(f"교통 데이터 로드/처리 중 오류 발생: {e}")

    # 상업 시설 데이터
    file_path_commercial = './data/서울시 상권분석서비스(집객시설-자치구).csv'
    seoul_commercial_facilities_df = pd.DataFrame()
    try:
        df_stores = pd.read_csv(file_path_commercial, encoding='cp949')
        average_stores_by_district = df_stores.groupby('자치구_코드_명')['집객시설_수'].mean().reset_index()
        merged_gdf_commercial = gdf_seoul_final_for_merge.merge(average_stores_by_district, on='자치구_코드_명', how='left')
        merged_gdf_commercial.dropna(subset=['집객시설_수'], inplace=True)
        merged_gdf_commercial['집객시설_수'] = merged_gdf_commercial['집객시설_수'].astype(int)
        seoul_commercial_facilities_df = merged_gdf_commercial[['자치구_코드_명', '집객시설_수']].copy()
    except Exception as e:
        st.warning(f"상업 시설 데이터 로드/처리 중 오류 발생: {e}")

    if not merged_population_density_df.empty:
        dashboard_data_df = merged_population_density_df[
            ['자치구_코드_명', '총_상주인구_수', '인구_밀도(명/km²)', '면적(km²)']
        ].copy()

        if not merged_bus_density_df.empty:
            dashboard_data_df = pd.merge(
                dashboard_data_df,
                merged_bus_density_df[['자치구_코드_명', '버스정류장_수', '버스정류장_밀도(개/km²)']],
                on='자치구_코드_명',
                how='inner'
            )

        if not merged_subway_density_df.empty:
            dashboard_data_df = pd.merge(
                dashboard_data_df,
                merged_subway_density_df[['자치구_코드_명', '지하철역_수', '지하철역_밀도(개/km²)']],
                on='자치구_코드_명',
                how='inner'
            )

        if not seoul_commercial_facilities_df.empty:
            dashboard_data_df = pd.merge(
                dashboard_data_df,
                seoul_commercial_facilities_df[['자치구_코드_명', '집객시설_수']],
                on='자치구_코드_명',
                how='inner'
            )

        if not seoul_transport_lacking_df.empty:
            dashboard_data_df = pd.merge(
                dashboard_data_df,
                seoul_transport_lacking_df[['자치구_코드_명', '정류장_부족_순위', '지하철_부족_순위', '종합_교통_부족_순위']],
                on='자치구_코드_명',
                how='inner'
            )
    else:
        st.error("인구 밀도 데이터가 없어 대시보드 데이터를 초기화할 수 없습니다.")
        dashboard_data_df = pd.DataFrame()

    return dashboard_data_df, geojson_data, gdf_seoul_for_map

dashboard_data_df, geojson_data, gdf_seoul_for_map = load_and_process_all_data()

column_display_names = {
    '자치구_코드_명': '자치구명',
    '총_상주인구_수': '총 상주인구 수',
    '인구_밀도(명/km²)': '인구 밀도 (명/km²)',
    '면적(km²)': '면적 (km²)',
    '버스정류장_수': '버스정류장 수',
    '버스정류장_밀도(개/km²)': '버스정류장 밀도 (개/km²)',
    '지하철역_수': '지하철역 수',
    '지하철역_밀도(개/km²)': '지하철역 밀도 (개/km²)',
    '집객시설_수': '집객시설 수',
    '정류장_부족_순위': '정류장 부족 순위',
    '지하철_부족_순위': '지하철 부족 순위',
    '종합_교통_부족_순위': '종합 교통 부족 순위',
}

st.set_page_config(layout="wide", page_title="서울시 도시계획 대시보드")
st.title("🏙️ 서울시 도시계획 및 대중교통 개선 대시보드")
st.markdown("서울시 25개 자치구의 인구, 상업시설, 버스/지하철 인프라 데이터를 분석하여 도시 계획 및 대중교통 개선 방안을 모색합니다.")

if dashboard_data_df.empty:
    st.error("데이터 로드에 실패했습니다. './data/' 폴더 안에 파일들이 있는지 확인해주세요.")
else:
    col1, col2 = st.columns(2)
    with col1:
        selected_district = st.selectbox(
            "자치구 선택:",
            options=['전체 구'] + sorted(dashboard_data_df['자치구_코드_명'].unique().tolist()),
            index=0
        )
    with col2:
        metric_options_map = {
            '인구 밀도 (명/km²)': '인구_밀도(명/km²)',
            '집객시설 수': '집객시설_수',
            '버스정류장 밀도 (개/km²)': '버스정류장_밀도(개/km²)',
            '지하철역 밀도 (개/km²)': '지하철역_밀도(개/km²)',
            '종합 교통 부족 순위': '종합_교통_부족_순위'
        }
        selected_metric_map_display = st.selectbox(
            "지도에 표시할 지표 선택:",
            options=list(metric_options_map.keys()),
            index=0
        )
        selected_metric_map = metric_options_map[selected_metric_map_display]

    st.markdown("---")
    st.subheader(f"🗺️ 서울시 자치구별 {selected_metric_map_display} 분포")

    filtered_df_for_map = dashboard_data_df.copy()
    if selected_district != '전체 구':
        filtered_df_for_map = filtered_df_for_map[filtered_df_for_map['자치구_코드_명'] == selected_district]

    center_lat, center_lon = 37.5665, 126.9780
    zoom_level = 9.5

    if selected_district != '전체 구':
        try:
            selected_gdf = gdf_seoul_for_map[gdf_seoul_for_map['자치구_코드_명'] == selected_district]
            if not selected_gdf.empty and selected_gdf.geometry.is_valid.all():
                centroid = selected_gdf.geometry.iloc[0].centroid
                center_lon, center_lat = centroid.x, centroid.y
                zoom_level = 11.5
        except Exception:
            pass

    colorscale = 'YlGnBu'
    if selected_metric_map == '종합_교통_부족_순위':
        colorscale = 'YlOrRd_r'

    fig_map = px.choropleth_mapbox(
        filtered_df_for_map,
        geojson=geojson_data,
        locations='자치구_코드_명',
        featureidkey='properties.자치구_코드_명',
        color=selected_metric_map,
        color_continuous_scale=colorscale,
        mapbox_style="carto-positron",
        zoom=zoom_level,
        center={"lat": center_lat, "lon": center_lon},
        opacity=0.7,
        labels={v: k for k, v in metric_options_map.items()},
        hover_name='자치구_코드_명'
    )
    fig_map.update_layout(margin={"r":0,"t":0,"l":0,"b":0}, height=600)
    st.plotly_chart(fig_map, use_container_width=True)

    st.markdown("---")
    st.subheader("📊 주요 지표 비교")
    
    col3, col4 = st.columns(2)
    with col3:
        metric_options_bar = {v: k for k, v in metric_options_map.items()}
        selected_metric_bar_display = st.selectbox("막대 차트 지표 선택:", options=list(metric_options_bar.values()), index=0)
        selected_metric_bar = [k for k, v in metric_options_bar.items() if v == selected_metric_bar_display][0]

    with col4:
         chart_type_selection = st.radio("표시 유형:", ['상위 10개', '하위 10개'], horizontal=True)

    sorted_df_bar = dashboard_data_df.sort_values(by=selected_metric_bar, ascending=(chart_type_selection == '하위 10개')).head(10)
    
    fig_bar = px.bar(
        sorted_df_bar,
        x='자치구_코드_명',
        y=selected_metric_bar,
        title=f"서울시 {selected_metric_bar_display} {chart_type_selection}",
        labels={'자치구_코드_명': '자치구명', selected_metric_bar: selected_metric_bar_display}
    )
    st.plotly_chart(fig_bar, use_container_width=True)

    st.markdown("---")
    st.subheader("📋 상세 데이터")
    st.dataframe(dashboard_data_df.rename(columns=column_display_names), use_container_width=True)
