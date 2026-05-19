import json
import re
import copy
from datetime import datetime
import streamlit as st
import pandas as pd
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# ==========================================
# CÁC HẰNG SỐ CỐ ĐỊNH HỆ THỐNG
# ==========================================
SO_NGAY_TOI_THIEU_VU = 15
NGUONG_GIAY_TUOI = 20
NGUONG_CHIA_SO = 10.0
TY_LE_CHIA = 100.0

MAP_CHI_SO = {
    'Số lần tưới': 'so_lan',
    'Số phút tưới': 'phut',
    'TBEC Thực tế': 'tbec',
    'EC Yêu cầu': 'ec_yc'
}

# ==========================================
# KHỐI XỬ LÝ LÕI & CACHE
# ==========================================
def json_decode_helper(file_bytes):
    try:
        noi_dung = file_bytes.decode("utf-8").strip()
        if not noi_dung: return []
        noi_dung = re.sub(r'}\s*{', '},{', noi_dung)
        if not noi_dung.startswith('['): noi_dung = f"[{noi_dung}]"
        noi_dung = re.sub(r',\s*]', ']', noi_dung)
        return json.loads(noi_dung)
    except Exception:
        return []

def chuan_hoa_so_thuc(gia_tri):
    if gia_tri is None or str(gia_tri).strip() == '': return 0.0
    try:
        so = float(gia_tri)
        return so / TY_LE_CHIA if so > NGUONG_CHIA_SO else so
    except ValueError:
        return 0.0

def lay_gia_tri_linh_hoat(dong, danh_sach_tu_khoa):
    for key, val in dong.items():
        if str(key).strip().lower() in danh_sach_tu_khoa:
            return val
    return None

def lay_du_lieu_ec_yeu_cau(du_lieu, stt_khu):
    ket_qua = {}
    stt_khu_chuan = str(stt_khu).strip().replace('.0', '')
    for dong in du_lieu:
        stt_dong = str(dong.get("STT", "")).strip().replace('.0', '')
        if stt_dong == stt_khu_chuan:
            tg_tho = lay_gia_tri_linh_hoat(dong, ['thời gian', 'thoi gian', 'thoi_gian'])
            ec = chuan_hoa_so_thuc(lay_gia_tri_linh_hoat(dong, ['ec yêu cầu', 'ec yeu cau', 'ec_yeu_cau']))
            if tg_tho and ec > 0:
                try:
                    tg_clean = str(tg_tho).split('.')[0].replace(':', '-')
                    ngay = datetime.strptime(tg_clean, "%Y-%m-%d %H-%M-%S").date()
                    if ngay not in ket_qua: ket_qua[ngay] = []
                    ket_qua[ngay].append(ec)
                except: pass
    return {n: round(sum(v)/len(v), 2) for n, v in ket_qua.items()}

@st.cache_data(show_spinner="Đang xử lý và bóc tách dữ liệu thô (Chỉ chạy 1 lần duy nhất)...")
def chuan_bi_du_lieu_tho(ng_bytes, cp_bytes, stt_khu):
    du_lieu_ng = json_decode_helper(ng_bytes)
    du_lieu_cp = json_decode_helper(cp_bytes)
    
    dict_ec_yc = lay_du_lieu_ec_yeu_cau(du_lieu_cp, stt_khu)
    ds_ban_ghi = []
    tap_ngay = set()
    stt_khu_chuan = str(stt_khu).strip().replace('.0', '')
    
    for dong in du_lieu_ng:
        stt_dong = str(dong.get("STT", "")).strip().replace('.0', '')
        if stt_dong == stt_khu_chuan:
            tg_tho = lay_gia_tri_linh_hoat(dong, ['thời gian', 'thoi gian', 'thoi_gian'])
            if tg_tho:
                try:
                    tg_clean = str(tg_tho).split('.')[0].replace(':', '-')
                    dt_obj = datetime.strptime(tg_clean, "%Y-%m-%d %H-%M-%S")
                    tap_ngay.add(dt_obj.date())
                    ds_ban_ghi.append({
                        'dt': dt_obj, 'ngay': dt_obj.date(),
                        'trang_thai': str(lay_gia_tri_linh_hoat(dong, ['trạng thái', 'trang thai'])).lower(),
                        'tbec': chuan_hoa_so_thuc(lay_gia_tri_linh_hoat(dong, ['tbec'])),
                        'tbph': chuan_hoa_so_thuc(lay_gia_tri_linh_hoat(dong, ['tbph']))
                    })
                except: pass
                
    if not tap_ngay: return []
    
    ds_ngay = sorted(list(tap_ngay))
    cac_vu = []
    vu_tam = [ds_ngay[0]]
    for i in range(1, len(ds_ngay)):
        if (ds_ngay[i] - ds_ngay[i-1]).days == 1:
            vu_tam.append(ds_ngay[i])
        else:
            cac_vu.append(vu_tam); vu_tam = [ds_ngay[i]]
    cac_vu.append(vu_tam)
    
    ds_ban_ghi.sort(key=lambda x: x['dt'])
    du_lieu_tong_hop = []
    
    for vu in cac_vu:
        if len(vu) < SO_NGAY_TOI_THIEU_VU: continue
        
        ket_qua_ngay_tho = []
        for thu_tu, ngay in enumerate(vu, 1):
            dong_trong_ngay = [r for r in ds_ban_ghi if r['ngay'] == ngay]
            giay_tong, so_lan, moc_bat = None, 0, None
            
            for r in dong_trong_ngay:
                if r['trang_thai'] == 'bật': moc_bat = r['dt']
                elif r['trang_thai'] == 'tắt' and moc_bat:
                    giay = (r['dt'] - moc_bat).total_seconds()
                    if giay > NGUONG_GIAY_TUOI:
                        giay_tong = (giay_tong or 0) + giay
                        so_lan += 1
                    moc_bat = None
            
            ds_ec = [r['tbec'] for r in dong_trong_ngay if r['tbec'] > 0]
            ds_ph = [r['tbph'] for r in dong_trong_ngay if r['tbph'] > 0]
            
            ket_qua_ngay_tho.append({
                'ngay': ngay,
                'nhan_ngay': f"N{thu_tu} ({ngay.strftime('%d/%m')})",
                'so_lan': so_lan,
                'phut': round((giay_tong or 0)/60, 1),
                'tbec': round(sum(ds_ec)/len(ds_ec), 2) if ds_ec else 0.0,
                'tbph': round(sum(ds_ph)/len(ds_ph), 2) if ds_ph else 0.0,
                'ec_yc': dict_ec_yc.get(ngay, 0.0)
            })
        du_lieu_tong_hop.append(ket_qua_ngay_tho)
        
    return du_lieu_tong_hop

# ==========================================
# THUẬT TOÁN ĐỘNG
# ==========================================
def tinh_toan_giai_doan_dong(danh_sach_ngay, khoa_chi_so, muc_sai_so):
    if not danh_sach_ngay: return []
    
    gd_hien_tai = 1
    gia_tri_chuan = -1.0
    
    for d in danh_sach_ngay:
        if float(d[khoa_chi_so]) > 0:
            gia_tri_chuan = float(d[khoa_chi_so])
            break
    if gia_tri_chuan == -1.0: gia_tri_chuan = 0.0

    for i in range(len(danh_sach_ngay)):
        ngay_hien_tai = danh_sach_ngay[i]
        gia_tri_nay = float(ngay_hien_tai[khoa_chi_so])
        
        if gia_tri_nay > 0:
            if abs(gia_tri_nay - gia_tri_chuan) > muc_sai_so:
                if i + 1 < len(danh_sach_ngay):
                    gia_tri_ngay_mai = float(danh_sach_ngay[i+1][khoa_chi_so])
                    if gia_tri_ngay_mai > 0 and abs(gia_tri_ngay_mai - gia_tri_nay) <= muc_sai_so:
                        gd_hien_tai += 1
                        gia_tri_chuan = gia_tri_nay
                        
        ngay_hien_tai['giai_doan'] = gd_hien_tai
        
    return danh_sach_ngay

# ==========================================
# GIAO DIỆN HIỂN THỊ (UI ĐƯỢC NÂNG CẤP)
# ==========================================
def xuat_bao_cao_streamlit(du_lieu_vu, stt_vu, chi_so_chon):
    so_ngay = len(du_lieu_vu)
    ngay_dau, ngay_cuoi = du_lieu_vu[0]['ngay'], du_lieu_vu[-1]['ngay']
    khoa_chi_so_input = MAP_CHI_SO[chi_so_chon]

    st.markdown(f"### 🌱 VỤ {stt_vu}: {ngay_dau.strftime('%d/%m/%Y')} ➔ {ngay_cuoi.strftime('%d/%m/%Y')} ({so_ngay} ngày)")
    
    nhan = [d['nhan_ngay'] for d in du_lieu_vu]
    ds_lan = [d['so_lan'] for d in du_lieu_vu]
    ds_tbec = [d['tbec'] if d['tbec'] > 0 else None for d in du_lieu_vu]
    ds_ec_yc = [d['ec_yc'] if d['ec_yc'] > 0 else None for d in du_lieu_vu]
    ds_giai_doan = [d['giai_doan'] for d in du_lieu_vu]

    fig = make_subplots(
        rows=2, cols=1, 
        shared_xaxes=True, 
        vertical_spacing=0.08,
        row_heights=[0.60, 0.40], 
        subplot_titles=("<b>📈 Biến động Số Lần Tưới & EC</b>", f"<b>Phân tách Giai đoạn: {chi_so_chon}</b>"),
        specs=[[{"secondary_y": True}], [{"secondary_y": False}]]
    )

    fig.add_trace(go.Bar(x=nhan, y=ds_lan, name='Số lần tưới', marker_color='#aed6f1', opacity=0.75), row=1, col=1, secondary_y=False)
    fig.add_trace(go.Scatter(x=nhan, y=ds_tbec, name='TBEC Thực', mode='lines', line=dict(color='#e74c3c', width=3)), row=1, col=1, secondary_y=True)
    fig.add_trace(go.Scatter(x=nhan, y=ds_ec_yc, name='EC Yêu Cầu', mode='lines', line=dict(color='#9b59b6', width=2.5, dash='dot')), row=1, col=1, secondary_y=True)

    giai_doan_duy_nhat = sorted(list(set(ds_giai_doan)))
    mau_sac_gd = ['#3498db', '#2ecc71', '#f1c40f', '#e67e22', '#9b59b6', '#34495e', '#1abc9c', '#e74c3c']
    
    for i, gd in enumerate(giai_doan_duy_nhat):
        x_gd = [d['nhan_ngay'] for d in du_lieu_vu if d['giai_doan'] == gd]
        y_gd = [d[khoa_chi_so_input] for d in du_lieu_vu if d['giai_doan'] == gd] 
        fig.add_trace(go.Bar(x=x_gd, y=y_gd, name=f'GĐ {gd} : {len(x_gd)} ngày', marker_color=mau_sac_gd[i % len(mau_sac_gd)]), row=2, col=1)

    fig.update_layout(
        height=900, 
        template="plotly_white", 
        hovermode="x unified", 
        margin=dict(l=30, r=30, t=60, b=20), 
        legend=dict(orientation="h", yanchor="bottom", y=1.03, xanchor="right", x=1, bgcolor='rgba(255,255,255,0.7)'), 
        barmode='group',
        font=dict(family="Arial, sans-serif", size=13, color="#2c3e50") 
    )
    
    fig.update_xaxes(tickfont=dict(color="black", size=12))
    fig.update_yaxes(title_text="<b>Số lần / Phút</b>", secondary_y=False, row=1, col=1, showgrid=True, gridcolor='#f1f2f6', tickfont=dict(color="black"))
    fig.update_yaxes(title_text="<b>Chỉ số EC</b>", secondary_y=True, row=1, col=1, showgrid=False, tickfont=dict(color="black"))
    fig.update_yaxes(title_text="<b>Giá trị GĐ</b>", row=2, col=1, showgrid=True, gridcolor='#f1f2f6', tickfont=dict(color="black"))

    st.plotly_chart(fig, use_container_width=True, config={'displayModeBar': False})

    st.markdown("#### 🔍 TỔNG KẾT CHI TIẾT THEO GIAI ĐOẠN")
    
    chon_gd = st.radio(
        f"✅ Tích chọn Giai đoạn của Vụ {stt_vu} để xem bảng dữ liệu:", 
        options=giai_doan_duy_nhat, 
        format_func=lambda x: f"Giai đoạn {x}",
        key=f"chon_gd_vu_{stt_vu}",
        horizontal=True
    )

    data_gd = [d for d in du_lieu_vu if d['giai_doan'] == chon_gd]
    
    if data_gd:
        df_gd = pd.DataFrame(data_gd)
        df_gd = df_gd[['ngay', 'nhan_ngay', 'so_lan', 'phut', 'tbec', 'tbph', 'ec_yc', 'giai_doan']]
        df_gd.rename(columns={
            'ngay': 'Ngày',
            'nhan_ngay': 'Nhãn Ngày',
            'so_lan': 'Số Lần Tưới',
            'phut': 'Số Phút',
            'tbec': 'TBEC Thực',
            'tbph': 'TBpH',
            'ec_yc': 'EC Yêu Cầu',
            'giai_doan': 'Giai Đoạn'
        }, inplace=True)

        so_ngay_gd = len(df_gd)
        tb_lan = df_gd['Số Lần Tưới'].mean()
        tb_phut = df_gd['Số Phút'].mean()
        tb_tbec = df_gd['TBEC Thực'].mean()
        tb_tbph = df_gd['TBpH'].mean()
        tb_ec_yc = df_gd['EC Yêu Cầu'].mean()

        df_summary = pd.DataFrame([{
            'Ngày': 'TRUNG BÌNH/TỔNG',
            'Nhãn Ngày': f'{so_ngay_gd} ngày',
            'Số Lần Tưới': round(tb_lan, 1),
            'Số Phút': round(tb_phut, 1),
            'TBEC Thực': round(tb_tbec, 2),
            'TBpH': round(tb_tbph, 2),
            'EC Yêu Cầu': round(tb_ec_yc, 2),
            'Giai Đoạn': chon_gd
        }])

        df_hien_thi = pd.concat([df_gd, df_summary], ignore_index=True)

        map_cot_hien_thi = {
            'Số lần tưới': 'Số Lần Tưới',
            'Số phút tưới': 'Số Phút',
            'TBEC Thực tế': 'TBEC Thực',
            'EC Yêu cầu': 'EC Yêu Cầu'
        }
        cot_to_dam = map_cot_hien_thi.get(chi_so_chon, '')

        def highlight_table(df):
            df_style = pd.DataFrame('', index=df.index, columns=df.columns)
            
            # TĂNG PHÔNG CHỮ LÊN 18px CÓ THÊM !IMPORTANT ĐỂ ĐẢM BẢO HIỂN THỊ
            css_dong_cuoi = 'font-weight: bold; background-color: #ecf0f1; color: #2c3e50; font-size: 18px !important;'
            df_style.iloc[-1, :] = css_dong_cuoi
            
            if cot_to_dam in df_style.columns:
                for idx in range(len(df_style) - 1): 
                    df_style.loc[idx, cot_to_dam] = 'font-weight: bold; color: #d35400; background-color: #fef9e7;'
                
                # Tăng kích thước chữ cho ô giao cắt
                css_giao_cat = 'font-weight: bold; background-color: #ecf0f1; color: #d35400; font-size: 18px !important;'
                df_style.iloc[-1, df_style.columns.get_loc(cot_to_dam)] = css_giao_cat
                
            return df_style

        format_mapping = {
            'Số Lần Tưới': lambda x: f"{x:g}" if isinstance(x, (int, float)) else x,
            'Số Phút': lambda x: f"{x:g}" if isinstance(x, (int, float)) else x,
            'TBEC Thực': lambda x: f"{x:g}" if isinstance(x, (int, float)) else x,
            'TBpH': lambda x: f"{x:g}" if isinstance(x, (int, float)) else x,
            'EC Yêu Cầu': lambda x: f"{x:g}" if isinstance(x, (int, float)) else x
        }

        st.caption(f"Chi tiết lịch sử tưới trong **Giai đoạn {chon_gd}** (Cột **{cot_to_dam}** đang được bôi sáng):")
        st.dataframe(df_hien_thi.style.format(format_mapping).apply(highlight_table, axis=None), use_container_width=True, hide_index=True)

# ==========================================
# KHỞI TẠO APP
# ==========================================
st.set_page_config(page_title="Hệ Thống Phân Tích", page_icon="📊", layout="wide")

if 'da_bat_dau' not in st.session_state:
    st.session_state['da_bat_dau'] = False

with st.sidebar:
    st.header("⚙️ TẢI FILE & CẤU HÌNH")
    
    file_ng = st.file_uploader("1. File: Lich nho giotj.json", type=['json'])
    file_cp = st.file_uploader("2. File: châm phân trung gian.json", type=['json'])
    
    st.divider()
    
    stt_khu = st.text_input("STT Khu (VD: 1, 2...):", value="1")
    ten_chi_so_chon = st.radio("Phân chia Giai đoạn theo:", options=list(MAP_CHI_SO.keys()), index=1)
    sai_so_nhap = st.number_input("Sai số cho phép:", value=2.00, step=0.10)
    
    if st.button("🚀 CHẠY BÁO CÁO TỔNG HỢP", type="primary", use_container_width=True):
        st.session_state['da_bat_dau'] = True

st.title("📊 HỆ THỐNG PHÂN TÍCH GIAI ĐOẠN ĐỘNG")
st.markdown("---")

if st.session_state['da_bat_dau']:
    if file_ng and file_cp:
        du_lieu_cac_vu_tho = chuan_bi_du_lieu_tho(file_ng.getvalue(), file_cp.getvalue(), stt_khu)
        
        if not du_lieu_cac_vu_tho:
            st.error(f"Không tìm thấy dữ liệu hợp lệ cho Khu {stt_khu}.")
        else:
            khoa_chi_so = MAP_CHI_SO[ten_chi_so_chon]
            
            for stt_vu, ket_qua_ngay_tho in enumerate(du_lieu_cac_vu_tho, 1):
                data_vu = copy.deepcopy(ket_qua_ngay_tho)
                ket_qua_gd = tinh_toan_giai_doan_dong(data_vu, khoa_chi_so, sai_so_nhap)
                xuat_bao_cao_streamlit(ket_qua_gd, stt_vu, ten_chi_so_chon)
                st.markdown("<br><br>", unsafe_allow_html=True)
                st.divider()
    else:
        st.info("👈 Hãy tải đủ 2 file JSON ở cột bên trái và bấm Chạy Báo Cáo nhé!")