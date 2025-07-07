from flask import Flask, request, jsonify
import pdfplumber
import re
import os

app = Flask(__name__)


def parse_wells_fargo(pdf):
    summary = {
        "bank": "Wells Fargo",
        "monthly_deposits": 0.0,
        "monthly_withdrawals": 0.0,
        "average_balance": 0.0,
        "ending_balance": 0.0,
        "nsf_count": 0,
        "low_balance_days": 0,
        "daily_balances": []
    }

    with pdfplumber.open(pdf) as pdf_file:
        text = "\n".join([page.extract_text() or "" for page in pdf_file.pages])

        deposits_match = re.search(r"Total electronic deposits/bank credits\s*\$([\d,]+\.\d{2})", text)
        if deposits_match:
            summary["monthly_deposits"] = float(deposits_match.group(1).replace(",", ""))

        debits_match = re.search(r"Total electronic debits/bank debits\s*\$([\d,]+\.\d{2})", text)
        if debits_match:
            summary["monthly_withdrawals"] = float(debits_match.group(1).replace(",", ""))

        avg_match = re.search(r"Average daily ledger balance \$([\d,]+\.\d{2})", text)
        if avg_match:
            summary["average_balance"] = float(avg_match.group(1).replace(",", ""))

        end_match = re.search(r"Ending balance\s*\$([\d,]+\.\d{2})", text)
        if end_match:
            summary["ending_balance"] = float(end_match.group(1).replace(",", ""))

        balance_lines = re.findall(r"(\d{2}/\d{2})\s+([\d,]+\.\d{2})", text)
        balances = [float(b.replace(",", "")) for _, b in balance_lines]
        summary["daily_balances"] = balances
        summary["low_balance_days"] = sum(1 for b in balances if b < 500)

        summary["nsf_count"] = len(re.findall(r"NSF|Returned Item", text, re.IGNORECASE))

    return summary


def parse_bofa(pdf):
    summary = {
        "bank": "Bank of America",
        "monthly_deposits": 0.0,
        "monthly_withdrawals": 0.0,
        "average_balance": 0.0,
        "ending_balance": 0.0,
        "nsf_count": 0,
        "low_balance_days": 0,
        "daily_balances": []
    }

    with pdfplumber.open(pdf) as pdf_file:
        text = "\n".join([page.extract_text() or "" for page in pdf_file.pages])

        deposits_match = re.search(r"Deposits and other credits\s+\$([\d,]+\.\d{2})", text)
        if deposits_match:
            summary["monthly_deposits"] = float(deposits_match.group(1).replace(",", ""))

        withdrawals_match = re.search(r"Total withdrawals and other debits\s*-\$([\d,]+\.\d{2})", text)
        if withdrawals_match:
            summary["monthly_withdrawals"] = float(withdrawals_match.group(1).replace(",", ""))

        checks_match = re.search(r"Total checks\s*-\$([\d,]+\.\d{2})", text)
        if checks_match:
            summary["monthly_withdrawals"] += float(checks_match.group(1).replace(",", ""))

        avg_match = re.search(r"Average ledger balance:\s*\$([\d,]+\.\d{2})", text)
        if avg_match:
            summary["average_balance"] = float(avg_match.group(1).replace(",", ""))

        end_match = re.search(r"Ending balance on .* \$([\d,]+\.\d{2})", text)
        if end_match:
            summary["ending_balance"] = float(end_match.group(1).replace(",", ""))

        balances = re.findall(r"\d{2}/\d{2}\s+([\d,]+\.\d{2})", text)
        balances = [float(b.replace(",", "")) for b in balances]
        summary["daily_balances"] = balances
        summary["low_balance_days"] = sum(1 for b in balances if b < 500)

        summary["nsf_count"] = len(re.findall(r"NSF|Returned Item", text, re.IGNORECASE))

    return summary


@app.route("/", methods=["GET"])
def home():
    return jsonify({
        "message": "Bank Statement Analyzer API",
        "endpoints": {
            "/analyze": "POST - Upload a PDF bank statement for analysis"
        }
    })

@app.route("/analyze", methods=["POST"])
def analyze():
    if 'file' not in request.files:
        return jsonify({"error": "No file part"}), 400
    file = request.files['file']
    filename = file.filename or ""

    if filename == '':
        return jsonify({"error": "No selected file"}), 400
    
    if not filename.lower().endswith('.pdf'):
        return jsonify({"error": "Please upload a PDF file"}), 400

    filepath = os.path.join("/tmp", filename)
    file.save(filepath)

    result = {}
    if "Wells" in filename:
        result = parse_wells_fargo(filepath)
    elif "Andrea" in filename or "BoA" in filename:
        result = parse_bofa(filepath)
    else:
        result = {"error": "Unknown bank format"}

    os.remove(filepath)
    return jsonify(result)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))

