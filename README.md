# seminatic_model
checking similarity between two json files 
import json
from openai import OpenAI

def compare_education_records(source_path, target_path, api_key):
    """Compare education records with enhanced data absence handling"""
    try:
        with open(source_path) as f:
            source = json.load(f)['resume_data']['person_of_interest']

        with open(target_path) as f:
            target = json.load(f)['person_of_interest']

        client = OpenAI(api_key=api_key)

        prompt = f"""Generate JSON comparison report following these rules:
        1. MATCH: Values are equivalent despite formatting/case differences
        2. NO_MATCH: Values meaningfully differ
        3. MATCH_TBD: Source has value but target is missing/null/empty
        4. MATCH_NA: Target has value but source is missing/null/empty
        5. RESULT_NA: Neither Target nor source has value

        Maintain this exact structure:
        {{
          "person_of_interest": {{
            "last_name": {{"comparison_result": "..."}},
            "first_name": {{"comparison_result": "..."}},
            "education": [
              {{
                "school_name": {{"comparison_result": "..."}},
                "degree_title": {{"comparison_result": "..."}},
                "dates_of_attendance": {{
                  "start_date": {{"comparison_result": "..."}},
                  "end_date": {{"comparison_result": "..."}}
                }},
                "date_of_degree_awarded": {{"comparison_result": "..."}},
                "major": [{{"comparison_result": "..."}}],
                "minor": [{{"comparison_result": "..."}}],
                "academic_honors": [{{"comparison_result": "..."}}],
                "honors_program": {{"comparison_result": "..."}}
              }}
            ]
          }}
        }}

        Source Data:
        {json.dumps(source, indent=2)}

        Target Data:
        {json.dumps(target, indent=2)}"""

        response = client.chat.completions.create(
            model="gpt-3.5",
            messages=[
                {"role": "system", "content": "You are a JSON comparison expert."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.1,
            response_format={"type": "json_object"}
        )

        raw_content = response.choices[0].message.content
        comparison_data = json.loads(raw_content)

        return merge_values(source, target, comparison_data)

    except Exception as e:
        print(f"Comparison error: {str(e)}")
        return None

def merge_values(source, target, comparison):
    """Merge data with RESULT_NA checks and certification format"""
    result = {"person_of_interest": {}}

    # Process personal info with RESULT_NA checks
    for field in ["last_name", "first_name"]:
        source_val = source.get(field, "")
        target_val = target.get(field, "")

        result["person_of_interest"][field] = {
            "value": source_val or target_val or "",
            "comparison_result": "RESULT_NA" if not source_val and not target_val
                else comparison["person_of_interest"].get(field, {}).get("comparison_result", "MATCH_TBD")
        }

    # Process education data
    result["person_of_interest"]["education"] = []
    max_edu = max(len(source.get("education", [])),
                 len(target.get("education", [])),
                 len(comparison["person_of_interest"].get("education", [])))

    for idx in range(max_edu):
        source_edu = source.get("education", [{}])[idx] if idx < len(source.get("education", [])) else {}
        target_edu = target.get("education", [{}])[idx] if idx < len(target.get("education", [])) else {}
        comp_edu = comparison["person_of_interest"].get("education", [{}])[idx] if idx < len(comparison["person_of_interest"].get("education", [])) else {}

        merged_edu = {
            "school_name": process_field(source_edu, target_edu, comp_edu, "school_name"),
            "degree_title": process_field(source_edu, target_edu, comp_edu, "degree_title"),
            "dates_of_attendance": process_dates(source_edu, target_edu, comp_edu),
            "date_of_degree_awarded": process_field(source_edu, target_edu, comp_edu, "date_of_degree_awarded"),
            "major": process_array(source_edu, target_edu, comp_edu, "major"),
            "minor": process_array(source_edu, target_edu, comp_edu, "minor"),
            "academic_honors": process_array(source_edu, target_edu, comp_edu, "academic_honors"),
            "honors_program": process_field(source_edu, target_edu, comp_edu, "honors_program")
        }
        result["person_of_interest"]["education"].append(merged_edu)

    return result

def process_field(source, target, comparison, field):
    source_val = source.get(field, "")
    target_val = target.get(field, "")
    return {
        "value": source_val or target_val or "",
        "comparison_result": "RESULT_NA" if not source_val and not target_val
            else comparison.get(field, {}).get("comparison_result", "MATCH_TBD")
    }

def process_dates(source, target, comparison):
    dates = {}
    for date_field in ["start_date", "end_date"]:
        source_val = source.get("dates_of_attendance", {}).get(date_field, "")
        target_val = target.get("dates_of_attendance", {}).get(date_field, "")
        dates[date_field] = {
            "value": source_val or target_val or "",
            "comparison_result": "RESULT_NA" if not source_val and not target_val
                else comparison.get("dates_of_attendance", {}).get(date_field, {}).get("comparison_result", "MATCH_TBD")
        }
    return dates

def process_array(source, target, comparison, field):
    merged = []
    source_items = source.get(field, [])
    target_items = target.get(field, [])
    comp_items = comparison.get(field, [])

    max_items = max(len(source_items), len(target_items), len(comp_items))

    for i in range(max_items):
        source_val = source_items[i] if i < len(source_items) else ""
        target_val = target_items[i] if i < len(target_items) else ""

        merged.append({
            "value": source_val or target_val or "",
            "comparison_result": "RESULT_NA" if not source_val and not target_val
                else comp_items[i].get("comparison_result", "MATCH_TBD") if i < len(comp_items) else "MATCH_TBD"
        })
    return merged

# Usage
API_KEY = " "
result = compare_education_records(
    "resume_data3.Json",
    "ground_truth3.json",
    API_KEY
)

if result:
    with open("certification-output.json", "w") as f:
        json.dump(result, f, indent=2, ensure_ascii=False)
