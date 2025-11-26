import sys
import os
import smtplib
import re
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import yfinance as yf
import pandas as pd
import requests
import io
from googletrans import Translator

# GitHub Secrets에서 환경변수 가져오기
GMAIL_USER = os.environ.get('GMAIL_USER')
GMAIL_PASSWORD = os.environ.get('GMAIL_PASSWORD')
TO_EMAIL = os.environ.get('TO_EMAIL')

def send_email(subject, html_content):
    if not GMAIL_USER or not GMAIL_PASSWORD or not TO_EMAIL:
        print("❌ 메일 설정(Secrets)이 부족하여 이메일을 보내지 않습니다.")
        return

    # 1. 정규표현식으로 이메일 주소만 추출 (쉼표, 따옴표, 공백 등 자동 제거)
    email_pattern = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
    recipients = re.findall(email_pattern, TO_EMAIL)

    if not recipients:
        print(f"❌ 유효한 이메일 주소를 찾을 수 없습니다. (입력값: {TO_EMAIL})")
        return

    print(f"DEBUG: 감지된 수신자 {len(recipients)}명 -> {recipients}")

    try:
        # Gmail SMTP 서버 연결 (한 번만 연결)
        server = smtplib.SMTP('smtp.gmail.com', 587)
        server.starttls()
        server.login(GMAIL_USER, GMAIL_PASSWORD)
        
        # [핵심 수정] 리스트를 통째로 넘기지 않고, 한 명씩 반복해서 발송
        success_count = 0
        for recipient in recipients:
            try:
                # 메시지 객체를 매번 새로 생성 (헤더 꼬임 방지)
                msg = MIMEMultipart()
                msg['From'] = GMAIL_USER
                msg['To'] = recipient  # 받는 사람에 해당 이메일만 표시
                msg['Subject'] = subject
                msg.attach(MIMEText(html_content, 'html'))
                
                # 전송 (recipient는 문자열)
                server.sendmail(GMAIL_USER, recipient, msg.as_string())
                print(f"  -> 전송 성공: {recipient}")
                success_count += 1
                
            except Exception as e_indiv:
                print(f"  -> 전송 실패 ({recipient}): {e_indiv}")

        server.quit()
        print(f"📧 전체 발송 완료! (성공: {success_count}/{len(recipients)})")
        
    except Exception as e:
        print(f"❌ SMTP 서버 연결 실패: {e}")

def scan_and_email():
    print("🚀 S&P 500 종목 스캔 시작... (개별 발송 모드)")
    
    translator = Translator()
    
    try:
        url = 'https://en.wikipedia.org/wiki/List_of_S%26P_500_companies'
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
        }
        response = requests.get(url, headers=headers)
        
        tables = pd.read_html(io.StringIO(response.text), match='Symbol')
        sp500_df = tables[0]
        
        sp500_df['Symbol'] = sp500_df['Symbol'].str.replace('.', '-', regex=False)
        ticker_to_name = dict(zip(sp500_df['Symbol'], sp500_df['Security']))
        tickers = sp500_df['Symbol'].tolist()
        print(f"✅ 총 {len(tickers)}개의 종목 리스트 확보")
        
    except Exception as e:
        print("❌ 리스트 확보 실패:", e)
        return

    print("📊 데이터 다운로드 중... (약 1~2분 소요)")
    data = yf.download(tickers, period="3mo", progress=False, auto_adjust=False)['Close']
    
    results = []
    
    for ticker in tickers:
        try:
            if ticker not in data.columns: continue
            
            series = data[ticker].dropna()
            if len(series) < 22: continue
            
            today_close = series.iloc[-1]
            past_20_max = series.iloc[-21:-1].max()
            past_21_max = series.iloc[-22:-1].max()
            
            if today_close > past_20_max and today_close <= past_21_max:
                
                prev_close = series.iloc[-2]
                pct_change = ((today_close - prev_close) / prev_close) * 100
                
                en_name = ticker_to_name.get(ticker, ticker)
                kr_name = en_name
                try:
                    translated = translator.translate(en_name, dest='ko')
                    kr_name = translated.text
                except:
                    pass
                
                results.append({
                    '티커': ticker,
                    '회사명': kr_name,
                    '현재가': f"${today_close:.2f}",
                    '등락률': f"{pct_change:.2f}%",
                    '21일고가(저항)': f"${max_21:.2f}"
                })
                
        except Exception:
            continue

    if len(results) > 0:
        df = pd.DataFrame(results)
        df = df[['티커', '회사명', '현재가', '등락률', '21일고가(저항)']]
        
        df['sort_key'] = df['등락률'].str.rstrip('%').astype(float)
        df = df.sort_values(by='sort_key', ascending=False)
        df = df.drop(columns=['sort_key'])
        
        html_table = df.to_html(index=False, border=1, justify='center', classes='table')
        
        email_body = f"""
        <html>
        <head>
            <style>
                table {{ border-collapse: collapse; width: 100%; }}
                th, td {{ padding: 8px; text-align: center; border: 1px solid #ddd; }}
                th {{ background-color: #f2f2f2; }}
                tr:nth-child(even) {{ background-color: #f9f9f9; }}
            </style>
        </head>
        <body>
            <h2>🚀 [알림] 오늘 포착된 종목 (20일 돌파 / 21일 저항)</h2>
            <p>S&P 500 종목 중 <b>총 {len(df)}개</b>가 조건에 부합합니다.</p>
            <br>
            {html_table}
            <br>
            <p><small>이 메일은 GitHub Actions에 의해 매일 아침 자동 발송됩니다.</small></p>
        </body>
        </html>
        """
        
        print(f"🎉 발견된 종목: {len(df)}개")
        send_email(f"[StockAlarm] {len(df)}개 종목 포착 (20일 신고가 필터)", email_body)
        
    else:
        print("💨 조건에 맞는 종목이 없습니다.")
        send_email("[StockAlarm] 오늘 포착된 종목 없음", "오늘은 조건(20일 돌파 / 21일 저항)에 맞는 종목이 없습니다.")

if __name__ == "__main__":
    scan_and_email()
