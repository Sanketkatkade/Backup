# Backup
# Backup

SO rates on Quotation

# API Server Script: get_last_sales_orders_for_items
# Expects: item_codes (string) like '["A","B"]' or 'A,B' or 'A'
# Sets: frappe.response["message"]

raw = frappe.form_dict.get('item_codes')
if not raw:
    frappe.response["message"] = {}
else:
    raw = raw.strip()
    items_list = []

    # parse simple list or comma-separated
    if len(raw) >= 2 and raw[0] == '[' and raw[-1] == ']':
        inner = raw[1:-1].strip()
        if inner != '':
            parts = inner.split(',')
            i = 0
            while i < len(parts):
                p = parts[i].strip()
                if len(p) >= 2 and ((p[0] == '"' and p[-1] == '"') or (p[0] == "'" and p[-1] == "'")):
                    p = p[1:-1]
                if p != '':
                    items_list.append(p)
                i = i + 1
    else:
        parts = raw.split(',')
        i = 0
        while i < len(parts):
            p = parts[i].strip()
            if len(p) >= 2 and ((p[0] == '"' and p[-1] == '"') or (p[0] == "'" and p[-1] == "'")):
                p = p[1:-1]
            if p != '':
                items_list.append(p)
            i = i + 1

    if not items_list:
        items_list = []

    sql_exact = """
        SELECT
            so.name as sales_order,
            so.customer_name as customer,
            so.transaction_date as transaction_date,
            soi.rate as rate,
            soi.item_code as soi_item_code
        FROM `tabSales Order Item` soi
        JOIN `tabSales Order` so ON so.name = soi.parent
        WHERE so.docstatus = 1
          AND TRIM(soi.item_code) = %s
        ORDER BY COALESCE(so.transaction_date, so.modified) DESC, so.modified DESC
        LIMIT 10
    """

    result = {}
    idx = 0
    while idx < len(items_list):
        item_code = items_list[idx]
        code = item_code
        if code is None:
            code = ''
        code = code.strip()

        rows = []
        try:
            rows = frappe.db.sql(sql_exact, (code,), as_dict=True)
        except Exception:
            rows = []

        simplified = []
        seen = {}
        c = 0
        j = 0
        while j < len(rows):
            r = rows[j]
            j = j + 1
            if not r:
                continue
            so_name = r.get('sales_order')
            if not so_name:
                continue
            if seen.get(so_name):
                continue
            td = r.get('transaction_date')
            td_str = None
            try:
                if td:
                    s = str(td)
                    if len(s) >= 10:
                        td_str = s[0:10]
                    else:
                        td_str = s
            except Exception:
                td_str = None

            simplified.append({
                "sales_order": so_name,
                "customer": r.get('customer'),
                "transaction_date": td_str,
                "rate": r.get('rate')
            })
            seen[so_name] = 1
            c = c + 1
            if c >= 3:
                break

        result[item_code] = simplified
        idx = idx + 1

    frappe.response["message"] = result





2) Item Code


import re
import frappe

def abbreviate(text):
    words = re.findall(r'\b\w+', text.upper())
    if len(words) >= 2:
        return ''.join(word[0] for word in words)[:3]
    else:
        return text.upper()[:3] if text else 'XXX'

if not doc.item_group or not doc.sub_group:
    frappe.throw("Both Item Group and Sub Group must be filled before saving.")

item_group_abbr = abbreviate(doc.item_group)
sub_group_abbr = abbreviate(doc.sub_group)

prefix = f"{item_group_abbr}-{sub_group_abbr}"
doc.name = frappe.model.naming.make_autoname(f"{prefix}-.###")



3) deal lost logic


if doc.sales_stage == "7. Lost":
    doc.status = "Lost"
    
if doc.sales_stage == "6. Won":
    doc.status = "Converted"
    
if doc.sales_stage == "8. Drop":
    doc.status = "Closed"


4) thereshold date logic

# Scheduler Event - Update Opportunities when threshold date has passed

today = frappe.utils.getdate(frappe.utils.nowdate())

# get candidates (not already closed or dropped)
opps = frappe.get_all(
    "Opportunity",
    filters=[
        ["docstatus", "!=", 2],
        ["status", "!=", "Closed"],
        ["sales_stage", "!=", "8. Drop"],
    ],
    fields=["name"],
    limit_page_length=1000
)

for o in opps:
    doc = frappe.get_doc("Opportunity", o.name)

    # only continue if threshold date is set
    if not doc.get("custom_threshold_date"):
        continue

    th_date = frappe.utils.getdate(doc.custom_threshold_date)

    if th_date and th_date <= today:
        doc.sales_stage = "8. Drop"
        doc.status = "Closed"
        try:
            doc.add_comment(
                "Info",
                f"System moved this Opportunity to '8. Drop' / Closed because custom_threshold_date ({th_date}) passed."
            )
        except Exception:
            # comment is optional, don’t block save
            pass
        doc.save()



5)profitability tab calculation

# Step 1: Set Revenue Total
doc.custom_revenue_total = doc.custom_total_inr or 0
custom_revenue_total = doc.custom_revenue_total or 1  # avoid divide by zero

# Step 2: Reset child tables
doc.custom_costing_b = []
doc.custom_nonoperating_cost_d = []

# Step 3: Material / Service Cost
if doc.custom_total_inr_1:
    amount = doc.custom_total_inr_1
    doc.append("custom_costing_b", {
        "b_costing": "Material/ Service Cost",
        "amount": amount,
        "of_revenue": (amount / custom_revenue_total) * 100
    })

# Step 4: Add Allowance Items as Operating Costs
for row in doc.custom_allowance1:
    # frappe.msgprint(f"Found Particular: {row.particulars}")
    if row.particulars == "Stay":
        doc.append("custom_costing_b", {
            "b_costing": "Accommodation",
            "amount": row.total_cost,
            "of_revenue": (row.total_cost / custom_revenue_total) * 100
        })
    elif row.particulars == "Food":
        doc.append("custom_costing_b", {
            "b_costing": "Food",
            "amount": row.total_cost,
            "of_revenue": (row.total_cost / custom_revenue_total) * 100
        })
    elif row.particulars == "Travel (train/flight/bus/car)":
        doc.append("custom_costing_b", {
            "b_costing": "Travel",
            "amount": row.total_cost,
            "of_revenue": (row.total_cost / custom_revenue_total) * 100
        })
    elif row.particulars == "Local Conveyance":
        doc.append("custom_costing_b", {
            "b_costing": "Local conveyance",
            "amount": row.total_cost,
            "of_revenue": (row.total_cost / custom_revenue_total) * 100
        })

# Step 5: Compute Total Operating Cost (B)
total_operating_cost_b = sum([r.amount for r in doc.custom_costing_b if r.amount])
doc.custom_total_operating_costsb = total_operating_cost_b
doc.custom_total__of_revenue = (total_operating_cost_b / custom_revenue_total) * 100

# Step 6: Operating Profit (C)
doc.custom_operating_profit_c = doc.custom_revenue_total - doc.custom_total_operating_costsb
doc.custom_operating_profit__ = (doc.custom_operating_profit_c / custom_revenue_total) * 100

# Step 7: Employee Cost (D)
doc.custom_employee_cost = doc.custom_total_employee_costing or 0
doc.custom__employee_cost = (doc.custom_employee_cost / custom_revenue_total) * 100

# Step 8: Non-operating Cost from custom_finance_working → populate custom_nonoperating_cost_d
for row in doc.custom_finance_working:
    amount = row.cost_amount or 0
    doc.append("custom_nonoperating_cost_d", {
        "non_operating": row.cost_name,
        "amount": amount,
        "revenue": (amount / custom_revenue_total) * 100
    })

# Step 9: Compute Total Operating Cost (D)
non_operating_total = sum([r.amount for r in doc.custom_nonoperating_cost_d if r.amount])
custom_total_operating_costs_d = doc.custom_employee_cost + non_operating_total
doc.custom_total_operating_costs_d = custom_total_operating_costs_d
doc.custom_total__of_revenue_d = (custom_total_operating_costs_d / custom_revenue_total) * 100

# Step 10: Net Profit Before Tax (E)
doc.custom_net_profit_before_tax_ecd = doc.custom_operating_profit_c - custom_total_operating_costs_d
doc.custom_total__of_revenue_e = (doc.custom_net_profit_before_tax_ecd / custom_revenue_total) * 100

# ✅ Step 10.5: Calculate Tax (F) using user-defined percentage `custom_tax_`
tax_percent = doc.custom_tax_ or 0
tax_amount = (tax_percent / 100) * doc.custom_net_profit_before_tax_ecd
doc.custom_tax_f = tax_amount

# Step 11: Net Profit After Tax (F)
doc.custom_net_profit_before_tax_gef = doc.custom_net_profit_before_tax_ecd - tax_amount
doc.custom_total__of_revenue_g = (doc.custom_net_profit_before_tax_gef / custom_revenue_total) * 100

# Step 12: Final Decision (custom_ideal_decision)
of_revenue_g = doc.custom_total__of_revenue_g
cost_of_capital = 0.0
if doc.custom_cost_of_capital__13:
    try:
        cost_of_capital = float(doc.custom_cost_of_capital__13)
    except ValueError:
        cost_of_capital = 0.0   # fallback if someone enters invalid data

if of_revenue_g < 10:
    doc.custom_ideal_decision = "Reject"
elif 10 <= of_revenue_g <= cost_of_capital:
    doc.custom_ideal_decision = "Hold"
else:
    doc.custom_ideal_decision = "Accept"



6) finance working and total


total = 0

for row in (doc.custom_finance_working or []):
    row.cost_amount = row.cost_amount or 0
    total = total + row.cost_amount

doc.custom_total_finance_working = total

# calculate difference of price
doc.custom_difference_of_price = (doc.total or 0) - (doc.custom_total_inr or 0)




7) Deal Stage Change

# Check if the status is changed to "Closed"
if doc.status == "Closed":
    # Set the sales_stage to "Drop"
    doc.sales_stage = "8. Drop"                       

if doc.workflow_state == "Rejected":
    # Set the sales_stage to "Drop" and close status
    doc.sales_stage = "8. Drop"
    doc.status = "Closed"

# Validate Custom Deal Date is not in the future
if doc.custom_deal_date:
    deal_date = frappe.utils.getdate(doc.custom_deal_date)
    today = frappe.utils.getdate(frappe.utils.today())
    if deal_date > today:
        frappe.throw("Custom Deal Date cannot be a future date.")


8) assign quotation to lead user

if doc.workflow_state == "Approved":
    for item in doc.items:
        if item.prevdoc_docname:
            opportunity_id = item.prevdoc_docname

            opp_owner = frappe.db.get_value("Opportunity", opportunity_id, "opportunity_owner")
            if opp_owner:
                # --- Assign (ToDo) ---
                already_assigned = frappe.db.exists(
                    "ToDo",
                    {
                        "reference_type": "Quotation",
                        "reference_name": doc.name,
                        "allocated_to": opp_owner,
                    },
                )
                if not already_assigned:
                    todo = frappe.new_doc("ToDo")
                    todo.allocated_to = opp_owner
                    todo.reference_type = "Quotation"
                    todo.reference_name = doc.name
                    todo.description = f"Quotation assigned from Opportunity {opportunity_id}"
                    todo.insert(ignore_permissions=True)

                # --- Share Document ---
                already_shared = frappe.db.exists(
                    "DocShare",
                    {
                        "share_doctype": "Quotation",
                        "share_name": doc.name,
                        "user": opp_owner,
                    },
                )
                if not already_shared:
                    share = frappe.new_doc("DocShare")
                    share.share_doctype = "Quotation"
                    share.share_name = doc.name
                    share.user = opp_owner
                    share.read = 1
                    share.write = 1
                    share.share = 1
                    share.insert(ignore_permissions=True)


9)text field child data replication


# Ensure exploded_items matches items length
for i, item_row in enumerate(doc.items):
    if len(doc.exploded_items) <= i:
        doc.append("exploded_items", {})

    doc.exploded_items[i].custom_small_text = item_row.custom_small_text
    doc.exploded_items[i].custom_long_text = item_row.custom_long_text


10)Item table append on one item


new_items = []

if doc.items:
    for item in doc.items:
        # Add original item
        new_items.append({
            "item_code": item.item_code,
            "item_name": item.item_name,
            "description": item.description,
            "uom": item.uom,
            "stock_uom": item.stock_uom,
            "schedule_date": item.schedule_date,
            "qty": item.qty or 1,
            "rate": item.rate or 0,
            "conversion_factor": item.conversion_factor or 1,
            "warehouse": item.warehouse
        })

        # Get Item master safely
        try:
            item_doc = frappe.get_doc("Item", item.item_code)
        except:
            continue

        # ✅ Replace hasattr with .get()
        sub_items = item_doc.get("custom_sub_item_table") or []

        for sub in sub_items:
            if not sub.item_code:
                continue

            try:
                sub_item = frappe.get_doc("Item", sub.item_code)
            except:
                continue

            new_items.append({
                "item_code": sub.item_code,
                "item_name": sub_item.item_name or "",
                "description": sub_item.description or "",
                "uom": sub_item.stock_uom or "",
                "stock_uom": sub_item.stock_uom or "",
                "schedule_date": item.schedule_date,
                "qty": 1,
                "rate": 0,
                "conversion_factor": 1,
                "warehouse": item.warehouse
            })

# Replace doc.items safely
if new_items:
    doc.set("items", [])
    for row in new_items:
        doc.append("items", row)



11) date fetch for child


# Ensure from_date and to_date are filled in the parent
if doc.from_date and doc.to_date:
    for row in doc.target:
        row.from_date = doc.from_date
        row.to_date = doc.to_date


12) costing emp fetch to crm table

# Server Script
# Doctype: Deal
# Trigger: Before Save

if doc.custom_resource == "Internal":
    doc.set("custom_employee", [])  # Clear old data

    total_costing = 0

    for row in doc.custom_internal_resource:
        if not row.employee:
            continue

        emp = frappe.db.get_value("Employee", row.employee, 
            ["employee_name", "custom_per_day_costing"], as_dict=True)

        if emp:
            try:
                per_day = float(emp.custom_per_day_costing or 0)
                days = float(row.days or 0)
            except:
                per_day = 0
                days = 0

            total = per_day * days

            doc.append("custom_employee", {
                "employee": row.employee,
                "employee_name": emp.employee_name,
                "per_day_costing": per_day,
                "number_of_days": days,
                "total": total
            })

            total_costing = total_costing + total

    doc.custom_total_employee_costing = total_costing

13) fetch month

# Server Script: Before Insert on Salary Slip

if doc.start_date:
    try:
        month_map = {
            "01": "January", "02": "February", "03": "March", "04": "April",
            "05": "May", "06": "June", "07": "July", "08": "August",
            "09": "September", "10": "October", "11": "November", "12": "December"
        }
        month_number = doc.start_date[5:7]
        doc.custom_month = month_map.get(month_number, "")
    except Exception as e:
        frappe.log_error(frappe.get_traceback(), "Set Custom Month Error")
        doc.custom_month = ""


14) Validate service expense

if doc.custom_purpose=="Service":
    for expense in doc.items:
        if doc.custom_purpose == "Service" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_service or 0
            service_budget = project.custom_service_budget or 0
            
            if current_total + expense.amount > service_budget:
                frappe.throw(
                    f"Food expense for project '{project.name}' exceeds the allocated budget. "
                    f"Current Total: {current_total}, Adding: {expense.amount}, Budget: {service_budget}"
                )


15) minus Service expense

if doc.custom_purpose =="Service":    
    for expense in doc.items:
        if doc.custom_purpose == "Service" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_service or 0
            project.custom_total_expense_service = max(0, current_total - expense.amount)
            project.save()

16) expense service

if doc.custom_purpose =="Service":    
    for expense in doc.items:
        if doc.custom_purpose == "Service" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_service or 0
            project.custom_total_expense_service = current_total + expense.amount
            project.save()


17) validate Supply expense

if doc.custom_purpose=="Supply":
    for expense in doc.items:
        if doc.custom_purpose == "Supply" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_supply or 0
            supply_budget = project.custom_supply_budget or 0
            
            if current_total + expense.amount > supply_budget:
                frappe.throw(
                    f"Food expense for project '{project.name}' exceeds the allocated budget. "
                    f"Current Total: {current_total}, Adding: {expense.amount}, Budget: {supply_budget}"
                )


18) minus supply expense

if doc.custom_purpose =="Supply":    
    for expense in doc.items:
        if doc.custom_purpose == "Supply" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_supply or 0
            project.custom_total_expense_supply = max(0, current_total - expense.amount)
            project.save()

19) Expense Supply

if doc.custom_purpose =="Supply":    
    for expense in doc.items:
        if doc.custom_purpose == "Supply" and doc.project:
            project = frappe.get_doc("Project", doc.project)
            current_total = project.custom_total_expense_supply or 0
            project.custom_total_expense_supply = current_total + expense.amount
            project.save()
20) flight book cancellation validate

if doc.workflow_state == "Cancelled" and not doc.reason_for_cancellation:
    frappe.throw("Reason for Cancellation is mandatory when the booking is cancelled.")




_______________________________________________________


client script



1) Fetching CRM Id and Prospect Name

frappe.ui.form.on("Quotation", {
  refresh(frm) {
    sync_crm_id(frm);
    sync_prospect_name(frm);
  },

  // CRM Id handlers
  custom_crm_id(frm) {             // Costing
    sync_crm_id(frm, "costing");
  },
  custom_crm_id_12(frm) {          // Profitability
    sync_crm_id(frm, "profitability");
  },
  custom_crm_id_details(frm) {     // Details
    sync_crm_id(frm, "details");
  },

  // Prospect Name handlers
  custom_prospect_name(frm) {            // Costing
    sync_prospect_name(frm, "costing");
  },
  custom_prospect_name_detail(frm) {     // Details
    sync_prospect_name(frm, "details");
  }
});

function sync_crm_id(frm, source) {
  let costing_val = frm.doc.custom_crm_id || "";
  let profitability_val = frm.doc.custom_crm_id_12 || "";
  let details_val = frm.doc.custom_crm_id_details || "";

  let master_val = "";
  if (source === "costing") master_val = costing_val;
  else if (source === "profitability") master_val = profitability_val;
  else if (source === "details") master_val = details_val;
  else master_val = costing_val || profitability_val || details_val;

  if (frm.doc.custom_crm_id !== master_val) {
    frm.set_value("custom_crm_id", master_val);
  }
  if (frm.doc.custom_crm_id_12 !== master_val) {
    frm.set_value("custom_crm_id_12", master_val);
  }
  if (frm.doc.custom_crm_id_details !== master_val) {
    frm.set_value("custom_crm_id_details", master_val);
  }
}

function sync_prospect_name(frm, source) {
  let costing_val = frm.doc.custom_prospect_name || "";
  let details_val = frm.doc.custom_prospect_name_detail || "";

  let master_val = "";
  if (source === "costing") master_val = costing_val;
  else if (source === "details") master_val = details_val;
  else master_val = costing_val || details_val;

  if (frm.doc.custom_prospect_name !== master_val) {
    frm.set_value("custom_prospect_name", master_val);
  }
  if (frm.doc.custom_prospect_name_detail !== master_val) {
    frm.set_value("custom_prospect_name_detail", master_val);
  }
}




2) Previous rate for SO


frappe.ui.form.on('Quotation', {
    onload: function(frm) {
        ensure_items_grid_button(frm);
    },
    refresh: function(frm) {
        ensure_items_grid_button(frm);
    }
});

function ensure_items_grid_button(frm) {
    const fieldname = 'items'; // change if your child-table fieldname differs

    if (!frm.fields_dict[fieldname] || !frm.fields_dict[fieldname].grid) {
        setTimeout(function() {
            if (frm && frm.fields_dict[fieldname] && frm.fields_dict[fieldname].grid) {
                ensure_items_grid_button(frm);
            }
        }, 250);
        return;
    }

    const wrapper = frm.fields_dict[fieldname].grid.wrapper;
    if ($(wrapper).find('.btn-show-rate-info').length) return;

    frm.fields_dict[fieldname].grid.add_custom_button(__('Show Rate Info'), function() {
        if (!frm.doc.items || frm.doc.items.length === 0) {
            frappe.msgprint(__('No items in this Quotation.'));
            return;
        }

        // collect distinct item codes preserving order
        const item_codes = [];
        (frm.doc.items || []).forEach(row => {
            if (row.item_code && item_codes.indexOf(row.item_code) === -1) {
                item_codes.push(row.item_code);
            }
        });

        if (item_codes.length === 0) {
            frappe.msgprint(__('No item codes found.'));
            return;
        }

        // call server API (Server Script name: get_last_sales_orders_for_items)
        frappe.call({
            method: 'get_last_sales_orders_for_items',
            args: { item_codes: JSON.stringify(item_codes) },
            freeze: true,
            freeze_message: __('Fetching sales order history...'),
            callback: function(resp) {
                console.log("Server response:", resp);
                const data = (resp && resp.message) ? resp.message : {};
                try {
                    // pass the current quotation items so we can show their custom text fields
                    render_sales_orders_dialog(data, item_codes, frm.doc.items || []);
                } catch (err) {
                    console.error("Error rendering dialog:", err, err.stack);
                    frappe.msgprint({ title: __('Error'), message: __('Failed to render sales order dialog. See console for details.') });
                }
            },
            error: function(xhr) {
                let msg = __('Error fetching data');
                try {
                    const server_messages = xhr.responseJSON && xhr.responseJSON._server_messages;
                    if (server_messages) {
                        const parsed = JSON.parse(server_messages);
                        if (parsed && parsed.length) msg = parsed[0];
                    } else if (xhr.responseJSON && xhr.responseJSON.message) {
                        msg = xhr.responseJSON.message;
                    }
                } catch (e) {}
                frappe.msgprint({ title: __('Error'), message: msg });
            }
        });

    }).addClass('btn-show-rate-info');
}

/* ---- Helper: safe number formatting (avoids frappe.format helpers) ---- */
function formatRate(val) {
    if (val === null || val === undefined || val === '') return '-';
    // allow numeric strings
    var n = Number(val);
    if (!isFinite(n)) {
        // fallback: show raw escaped value
        return frappe.utils.escape_html(String(val));
    }
    // integer -> show without decimals
    if (Math.floor(n) === n) return String(n);
    // non-integer -> show up to 6 decimal places, trim trailing zeros
    var s = n.toLocaleString(undefined, { maximumFractionDigits: 6 });
    // remove trailing zeros and possible trailing decimal separator
    s = s.replace(/(?:\.0+|(\.\d+?)0+)$/, '$1');
    return s;
}

/* ---- Renderer ----
   data_map: server response keyed by item_code -> array of rows (sales order history)
   item_codes: ordered array of item codes
   items_rows: the child table rows from the current Quotation (so we can pull small/long text)
*/
function render_sales_orders_dialog(data_map, item_codes, items_rows) {
    const normalized = {};
    Object.keys(data_map || {}).forEach(function(k) {
        normalized[String(k)] = data_map[k];
    });

    // build map item_code -> first matching child-row's custom text fields
    const item_texts = {};
    (items_rows || []).forEach(function(row) {
        const code = row.item_code ? String(row.item_code) : null;
        if (!code) return;
        if (!item_texts[code]) {
            item_texts[code] = {
                small: row.custom_small_text || '',
                long: row.custom_long_text || ''
            };
        }
        // we keep first occurrence only (preserve order); if you want concat for duplicates, change logic here
    });

    let html = '<div style="max-height:60vh; overflow:auto;">';
    html += '<table class="table table-bordered" style="width:100%; table-layout:fixed; word-wrap:break-word;">';
    html += '<thead><tr>' +
            '<th style="width:12%;">Item Code</th>' +
            '<th style="width:16%;">Small Text</th>' +
            '<th style="width:22%;">Long Text</th>' +
            '<th style="width:18%;">Customer</th>' +
            '<th style="width:12%;">Sales Order</th>' +
            '<th style="width:10%;">Transaction Date</th>' +
            '<th style="width:10%;">Rate</th>' +
            '</tr></thead><tbody>';

    const render_keys = Array.isArray(item_codes) && item_codes.length ? item_codes.map(String) : Object.keys(normalized);
    const remaining_keys = Object.keys(normalized).filter(function(k) { return render_keys.indexOf(k) === -1; });
    const all_keys = render_keys.concat(remaining_keys);

    if (all_keys.length === 0) {
        html += '<tr><td colspan="7">' + frappe.utils.escape_html(__('No data returned')) + '</td></tr>';
    } else {
        all_keys.forEach(function(item_code) {
            var rows = Array.isArray(normalized[item_code]) ? normalized[item_code] : [];
            // get text values (may be empty)
            var texts = item_texts[item_code] || { small: '', long: '' };
            var small_html = texts.small ? '<div style="white-space:pre-wrap; max-height:60px; overflow:auto;">' + frappe.utils.escape_html(texts.small) + '</div>' : '-';
            var long_html  = texts.long  ? '<div style="white-space:pre-wrap; max-height:120px; overflow:auto;">' + frappe.utils.escape_html(texts.long) + '</div>'  : '-';

            if (!rows.length) {
                html += '<tr>' +
                        '<td>' + frappe.utils.escape_html(item_code) + '</td>' +
                        '<td>' + small_html + '</td>' +
                        '<td>' + long_html + '</td>' +
                        '<td colspan="4">' + frappe.utils.escape_html(__('No confirmed sales orders found')) + '</td>' +
                        '</tr>';
            } else {
                rows.forEach(function(r, idx) {
                    var customer = r && r.customer ? r.customer : '-';
                    var so = r && r.sales_order ? r.sales_order : '-';
                    var td = r && r.transaction_date ? r.transaction_date : '-';
                    var rate_val = (r && (typeof r.rate === 'number' || !isNaN(Number(r.rate)))) ? r.rate : null;

                    var so_link = '-';
                    if (so !== '-') {
                        // create an in-app link (use data-so and class so clicking goes through frappe.set_route)
                        var so_enc = encodeURIComponent(so);
                        so_link = "<a href=\"#\" class=\"so-link\" data-so=\"" + so_enc + "\" title=\"" + frappe.utils.escape_html(so) + "\">" + frappe.utils.escape_html(so) + "</a>";
                    }

                    // for subsequent rows of same item_code, don't repeat the small/long text (keep only for first row) to avoid visual duplication
                    var small_cell = (idx === 0) ? small_html : '';
                    var long_cell  = (idx === 0) ? long_html  : '';

                    html += '<tr>' +
                            '<td>' + frappe.utils.escape_html(item_code) + '</td>' +
                            '<td>' + small_cell + '</td>' +
                            '<td>' + long_cell + '</td>' +
                            '<td>' + frappe.utils.escape_html(customer) + '</td>' +
                            '<td>' + so_link + '</td>' +
                            '<td>' + frappe.utils.escape_html(td) + '</td>' +
                            '<td style="text-align:right;">' + (rate_val !== null ? formatRate(rate_val) : '-') + '</td>' +
                            '</tr>';
                });
            }
        });
    }

    html += '</tbody></table></div>';

    const d = new frappe.ui.Dialog({
        title: __('Last Sales Orders'),
        fields: [{ fieldtype: 'HTML', fieldname: 'history_html' }],
        size: 'large'
    });

    d.fields_dict.history_html.wrapper.innerHTML = html;

    d.show();

    // After show, tweak modal dialog width so table has space.
    try {
        d.$wrapper.find('.modal-dialog').css({'max-width': '1100px', 'width': '95%'});
        d.$wrapper.find('a.so-link').css({'cursor': 'pointer', 'text-decoration': 'underline'});
    } catch (e) {
        console.warn('Could not set dialog width styling', e);
    }

    // Click handler for Sales Order links: close dialog and route to Sales Order form
    d.$wrapper.on('click', 'a.so-link', function(e) {
        e.preventDefault();
        var so_enc = $(this).attr('data-so') || '';
        var so_name = decodeURIComponent(so_enc);
        d.hide();
        frappe.set_route('Form', 'Sales Order', so_name);
    });

    d.set_primary_action(__('Close'), function() { d.hide(); });
}


3) Item code

frappe.ui.form.on('Item', {
    before_save: function(frm) {
        if (!frm.doc.item_code && frm.doc.item_group && frm.doc.custom_sub_group) {

            function abbreviate(text) {
                if (!text) return 'XXX';
                let words = text.trim().toUpperCase().split(/\s+/);
                if (words.length >= 2) {
                    return words.map(w => w[0]).join('').substring(0, 3);
                }
                return words[0].substring(0, 3);
            }

            const ig_abbr = abbreviate(frm.doc.item_group);
            const sg_abbr = abbreviate(frm.doc.custom_sub_group);
            const prefix = `${ig_abbr}-${sg_abbr}-`;

            frappe.call({
                method: "frappe.client.get_list",
                args: {
                    doctype: "Item",
                    fields: ["item_code"],
                    filters: [
                        ["item_code", "like", prefix + "%"]
                    ],
                    order_by: "creation desc",
                    limit_page_length: 1
                },
                callback: function(r) {
                    let lastNumber = 0;

                    if (r.message && r.message.length > 0) {
                        let lastCode = r.message[0].item_code;
                        let parts = lastCode.split("-");
                        let numPart = parts[parts.length - 1];
                        if (!isNaN(numPart)) {
                            lastNumber = parseInt(numPart);
                        }
                    }

                    const nextNumber = lastNumber + 1;
                    const formattedNumber = nextNumber.toString().padStart(3, '0');
                    const newItemCode = prefix + formattedNumber;

                    frm.set_value('item_code', newItemCode);
                }
            });
        }
    }
});


4) item 1

frappe.ui.form.on('Item', {
    before_save: function(frm) {
        if (frm.doc.item_group && frm.doc.custom_sub_group && !frm.doc.naming_series) {
            function abbreviate(text) {
                if (!text) return 'XXX';
                let words = text.trim().toUpperCase().split(/\s+/);
                if (words.length >= 2) {
                    return words.map(w => w[0]).join('').substring(0, 3);
                }
                return words[0].substring(0, 3);
            }

            const ig_abbr = abbreviate(frm.doc.item_group);
            const sg_abbr = abbreviate(frm.doc.custom_sub_group);
            const prefix = `${ig_abbr}-${sg_abbr}-.`;

            // Check if prefix exists in naming series options
            if (frm.fields_dict.naming_series.df.options.includes(prefix)) {
                frm.set_value('naming_series', prefix);
            } else {
                frappe.msgprint(`Naming series ${prefix} not found. Please add it in Naming Series settings.`);
            }
        }
    }
});


5) HSN/Code Bypass

frappe.ui.form.on('Item', {
    onload: function(frm) {
        frm.set_df_property('hsn_sac', 'reqd', false);
    }
});



6) custom MR Btn


frappe.ui.form.on("Opportunity", {
    refresh: function (frm) {
        // Clear existing buttons to avoid duplicates
        // frm.clear_custom_buttons();

        // Allowed workflow states
        const allowed_states = [
            "Completed Technical Evaluation",
            "Completed RFQ",
            "Pending For RFQ"
        ];

        if (allowed_states.includes(frm.doc.workflow_state)) {
            frm.add_custom_button(__('Material Request'), function () {
                frappe.new_doc("Material Request", {
                    custom_deal_id: frm.doc.name  // link Opportunity name
                });
            }, __("Create"));
        }
    }
});



7) total calculate for target main


// Client Script for Doctype: Target (no IIFE wrapper)
// Sums child table field `target` (currency) from child table fieldname: category_target
// and writes the result into parent field `total_target`

frappe.ui.form.on('Target Main', {
    onload: function(frm) {
        calculate_total(frm);
    },
    refresh: function(frm) {
        calculate_total(frm);
    },
    quarter_target_add: function(frm, cdt, cdn) {
        calculate_total(frm);
    },
    quarter_target_remove: function(frm, cdt, cdn) {
        calculate_total(frm);
    }
});

frappe.ui.form.on('Quarter child', {
    // fired when any row's `target` value changes
    target_amount: function(frm, cdt, cdn) {
        calculate_total(frm);
    }
});

function calculate_total(frm) {
    let total_inr = 0;

    (frm.doc.quarter_target || []).forEach(row => {
        total_inr += flt(row.target_amount);
    });

    frm.set_value('yearly_total_target', total_inr);
}



8) total calculate for target

// Client Script for Doctype: Target (no IIFE wrapper)
// Sums child table field `target` (currency) from child table fieldname: category_target
// and writes the result into parent field `total_target`

frappe.ui.form.on('Target', {
    onload: function(frm) {
        calculate_total(frm);
    },
    refresh: function(frm) {
        calculate_total(frm);
    },
    target_add: function(frm, cdt, cdn) {
        calculate_total(frm);
    },
    target_remove: function(frm, cdt, cdn) {
        calculate_total(frm);
    }
});

frappe.ui.form.on('Category-Target', {
    // fired when any row's `target` value changes
    target: function(frm, cdt, cdn) {
        calculate_total(frm);
    }
});

function calculate_total(frm) {
    let total_inr = 0;

    (frm.doc.target || []).forEach(row => {
        total_inr += flt(row.target);
    });

    frm.set_value('total_target', total_inr);
}


9) finance working calculation

function compute_basis(frm) {
    // include custom_total_cost_by_customer only when checkbox is ticked
    return (
        flt(frm.doc.custom_total_employee_costing) +
        flt(frm.doc.custom_total_inr_1) +
        flt(frm.doc.custom_total_cost_by_company) +
        (frm.doc.custom_add_in_basic_amount ? flt(frm.doc.custom_total_cost_by_customer) : 0)
    );
}

function update_finance_working_rows(frm) {
    const basis = compute_basis(frm);

    (frm.doc.custom_finance_working || []).forEach(row => {
        row.basis_amount = basis;
        row.cost_amount = flt(row.basis_amount) * flt(row.cost_) / 100;
    });

    frm.refresh_field('custom_finance_working');
}

// Parent Doctype: Quotation
frappe.ui.form.on('Quotation', {
    validate(frm) {
        update_finance_working_rows(frm);
    },

    custom_total_employee_costing(frm) {
        update_finance_working_rows(frm);
    },

    custom_total_inr_1(frm) {
        update_finance_working_rows(frm);
    },

    custom_total_cost_by_company(frm) {
        update_finance_working_rows(frm);
    },

    // new: when customer cost changes, recalc
    custom_total_cost_by_customer(frm) {
        update_finance_working_rows(frm);
    },

    // new: when checkbox toggled, recalc
    custom_add_in_basic_amount(frm) {
        update_finance_working_rows(frm);
    },

    custom_finance_working_on_form_rendered(frm) {
        // triggered when child table rows are rendered (optional for UX)
        update_finance_working_rows(frm);
    }
});

// Child Table: Finance Working
frappe.ui.form.on('Finance Working', {
    cost_(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        const basis = compute_basis(frm);

        row.basis_amount = basis;
        row.cost_amount = flt(basis) * flt(row.cost_) / 100;

        frm.refresh_field('custom_finance_working');
    },

    custom_finance_working_add(frm) {
        update_finance_working_rows(frm);
    },

    custom_finance_working_remove(frm) {
        update_finance_working_rows(frm);
    }
});


10) crm id replicate to profitability

frappe.ui.form.on('Quotation', {
    // When the field is changed in the form
    custom_crm_id: function(frm) {
        // copy the value (handles null/undefined as well)
        frm.set_value('custom_crm_id_12', frm.doc.custom_crm_id || null);
    }

   
});




11) create btn hide on workflow


frappe.ui.form.on('Opportunity', {
    refresh(frm) {
        setTimeout(() => {
            if (!["Completed Technical Evaluation", "Completed RFQ","Pending For RFQ"].includes(frm.doc.workflow_state)) {
                frm.remove_custom_button('Supplier Quotation', 'Create');
                frm.remove_custom_button('Request For Quotation', 'Create');
                frm.remove_custom_button('Quotation', 'Create');
                frm.remove_custom_button('Customer', 'Create');
            }
        }, 500); // delay so buttons are rendered before removal
    },

    workflow_state(frm) {
        frm.trigger("refresh");
    }
});



12) Set Date on the basis of Sales Stage


frappe.ui.form.on("Opportunity", {
    sales_stage: function(frm) {
        let today = frappe.datetime.nowdate();

        if (frm.doc.sales_stage === "1. Identified Opportunity") {
            frm.set_value("custom_identified_opportunity_date", today);
        }
        else if (frm.doc.sales_stage === "2. Qualified Opportunity") {
            frm.set_value("custom_qualified_opportunity_date", today);
        }
        else if (frm.doc.sales_stage === "3. Create Proposal") {
            frm.set_value("custom_create_proposal_date", today);
        }
        else if (frm.doc.sales_stage === "4. Proposal Submitted") {
            frm.set_value("custom_proposal_submitted_date", today);
        }
        else if (frm.doc.sales_stage === "5. Under Negotiation") {
            frm.set_value("custom_under_negotiation", today);
        }
        else if (frm.doc.sales_stage === "6. Won") {
            frm.set_value("custom_won_date", today);
            frm.set_value("custom_lost_date", today);
        }
        else if (frm.doc.sales_stage === "7. Lost") {
            frm.set_value("custom_lost_date_", today);
            frm.set_value("custom_lost_date", today);
        }
        else if (frm.doc.sales_stage === "8. Drop") {
            frm.set_value("custom_drop_date", today);
        }
    }
});



13) get items from BOM


frappe.ui.form.on('Material Request', {
    refresh: function(frm) {
        // Add custom button in the top bar
        frm.add_custom_button(__('Custtom Get Items from BOM'), function() {
            get_items_from_bom(frm);
        }, __('Get Items'));
    }
});

// Function to fetch items from BOM
function get_items_from_bom(frm) {
    var d = new frappe.ui.Dialog({
        title: __("Get Items from BOM"),
        fields: [
            {
                fieldname: "bom",
                fieldtype: "Link",
                label: __("BOM"),
                options: "BOM",
                reqd: 1,
                get_query: function () {
                    return { filters: { docstatus: 1, is_active: 1 } };
                },
            },
            {
                fieldname: "warehouse",
                fieldtype: "Link",
                label: __("For Warehouse"),
                options: "Warehouse",
                reqd: 1,
            },
            { 
                fieldname: "qty", 
                fieldtype: "Float", 
                label: __("Quantity"), 
                reqd: 1, 
                default: 1 
            },
            {
                fieldname: "fetch_exploded",
                fieldtype: "Check",
                label: __("Fetch exploded BOM (including sub-assemblies)"),
                default: 1,
            },
        ],
        primary_action_label: __("Get Items"),
        primary_action(values) {
            if (!values) return;
            values["company"] = frm.doc.company;
            if (!frm.doc.company) frappe.throw(__("Company field is required"));

            frappe.call({
                method: "erpnext.manufacturing.doctype.bom.bom.get_bom_items",
                args: values,
                callback: function (r) {
                    if (!r.message) {
                        frappe.throw(__("BOM does not contain any stock item"));
                    } else {
                        // Remove empty first row if exists
                        erpnext.utils.remove_empty_first_row(frm, "items");

                        // Add items to child table
                        $.each(r.message, function (i, item) {
                            var row = frappe.model.add_child(frm.doc, "Material Request Item", "items");

                            row.item_code = item.item_code;
                            row.item_name = item.item_name;
                            row.description = item.description;
                            row.warehouse = values.warehouse;
                            row.uom = item.stock_uom;
                            row.stock_uom = item.stock_uom;
                            row.conversion_factor = 1;
                            row.qty = item.qty;
                            row.project = item.project;

                            // Custom fields from BOM
                            row.custom_small_text = item.custom_small_text || "";
                            row.custom_long_text = item.custom_long_text || "";
                        });
                    }
                    d.hide();
                    refresh_field("items");
                },
            });
        },
    });

    d.show();
}


14) sub item add
frappe.ui.form.on('Purchase Order Item', {
    item_code: function (frm, cdt, cdn) {
        let row = locals[cdt][cdn];
        if (!row.item_code) return;

        frappe.db.get_doc('Item', row.item_code).then(itemDoc => {
            if (itemDoc.custom_sub_item_table && itemDoc.custom_sub_item_table.length > 0) {
                itemDoc.custom_sub_item_table.forEach(subRow => {
                    // Add sub-item row
                    frappe.call({
                        method: "frappe.client.get",
                        args: {
                            doctype: "Item",
                            name: subRow.item_code
                        },
                        callback: function (r) {
                            if (r.message) {
                                let item = r.message;
                                let child = frm.add_child('items');
                                child.item_code = item.name;
                                child.item_name = item.item_name;
                                child.description = item.description;
                                child.uom = item.stock_uom;
                                child.schedule_date = frm.doc.schedule_date || frappe.datetime.nowdate();

                                // Optional: set qty as 1 by default
                                child.qty = 1;

                                frm.refresh_field('items');
                            }
                        }
                    });
                });
            }
        });
    }
});



15) link filter options
frappe.ui.form.on('Target', {
    validate: function(frm) {
        const doctype_map = {
            'Planning': 'Product and Service for Planning',
            'Design': 'Product and Service for Design',
            'Implementation': 'Product and Service for Implementation',
            'Audit & Compliance': 'Product and Service for Audit and Compliance',
            'EPC': 'Product and Service for EPC'
        };

        (frm.doc.target || []).forEach(row => {
            if (row.service_category) {
                row.link_doctype = doctype_map[row.service_category] || '';
            } else {
                row.link_doctype = '';
            }

            // optional: clear productservice_category if link_doctype is empty
            if (!row.link_doctype) {
                row.productservice_category = '';
            }
        });
    }
});

frappe.ui.form.on('Category-Target', {
    service_category(frm, cdt, cdn) {
        const child = locals[cdt][cdn];
        const doctype_map = {
            'Planning': 'Product and Service for Planning',
            'Design': 'Product and Service for Design',
            'Implementation': 'Product and Service for Implementation',
            'Audit & Compliance': 'Product and Service for Audit and Compliance',
            'EPC': 'Product and Service for EPC'
        };

        frappe.model.set_value(cdt, cdn, 'link_doctype', doctype_map[child.service_category] || '');
        frappe.model.set_value(cdt, cdn, 'productservice_category', null);
    }
});


16) dynamic link for table


frappe.ui.form.on('Target', {
    onload: function(frm) {
        frm.fields_dict['target'].grid.get_field('productservice_category').get_query = function (doc, cdt, cdn) {
            const row = locals[cdt][cdn];

            const doctype_map = {
                'Planning': 'Product and Service for Planning',
                'Design': 'Product and Service for Design',
                'Implementation': 'Product and Service for Implementation',
                'Audit & Compliance': 'Product and Service for Audit and Compliance',
                'EPC': 'Product and Service for EPC'
            };

            const target_doctype = doctype_map[row.service_category];

            if (!target_doctype) {
                frappe.msgprint("Please select a valid Service Category first.");
                return false;
            }

            return {
                query: "frappe.desk.search.search_link",
                filters: {
                    doctype: target_doctype
                }
            };
        };
    }
});


17) total employee calculation



frappe.ui.form.on('Quotation', {
    // Trigger on form load and on any child table update
    refresh: function(frm) {
        calculate_total_internal_costing(frm);
    }
});

// Trigger calculations when number_of_days or per_day_costing changes
frappe.ui.form.on('Internal Resource Costing', {
    number_of_days: function(frm, cdt, cdn) {
        calculate_row_total_and_update(frm, cdt, cdn);
    },
    per_day_costing: function(frm, cdt, cdn) {
        calculate_row_total_and_update(frm, cdt, cdn);
    }
});

// Calculate total per row and update parent field
function calculate_row_total_and_update(frm, cdt, cdn) {
    let row = locals[cdt][cdn];
    row.total = (parseFloat(row.number_of_days) || 0) * (parseFloat(row.per_day_costing) || 0);
    frm.refresh_field('custom_employee');
    calculate_total_internal_costing(frm);
}

// Calculate sum of all row totals
function calculate_total_internal_costing(frm) {
    let total_sum = 0;
    (frm.doc.custom_employee || []).forEach(row => {
        total_sum += parseFloat(row.total) || 0;
    });
    frm.set_value('custom_total_employee_costing', total_sum);
}



18) total calculate for cost table

frappe.ui.form.on('Quotation', {
    refresh(frm) {
        calculate_total_inr(frm);
    },

    custom_cost_detail_add: function(frm) {
        calculate_total_inr(frm);
    },

    custom_cost_detail_remove: function(frm) {
        calculate_total_inr(frm);
    }
});

frappe.ui.form.on('Cost details', {
    amount_inr: function(frm, cdt, cdn) {
        calculate_total_inr(frm);
    }
});

function calculate_total_inr(frm) {
    let total_inr = 0;

    (frm.doc.custom_cost_detail || []).forEach(row => {
        total_inr += flt(row.amount_inr);
    });

    frm.set_value('custom_total_inr_1', total_inr);
}



19) total calculate for revenue

frappe.ui.form.on('Quotation', {
    // Trigger calculation on refresh and after saving
    refresh: function(frm) {
        calculate_revenue_totals(frm);
    },

    // Trigger calculation when any row is added or removed
    custom_revenue_detail_add: function(frm) {
        calculate_revenue_totals(frm);
    },
    custom_revenue_detail_remove: function(frm) {
        calculate_revenue_totals(frm);
    }
});

// Trigger when any field inside the child table is changed
frappe.ui.form.on('Revenue details', {
    amount: function(frm, cdt, cdn) {
        calculate_revenue_totals(frm);
    },
    amount_inr: function(frm, cdt, cdn) {
        calculate_revenue_totals(frm);
    }
});

// Helper function to calculate totals
function calculate_revenue_totals(frm) {
    let total_transaction_currency = 0;
    let total_inr = 0;

    (frm.doc.custom_revenue_detail || []).forEach(row => {
        total_transaction_currency += flt(row.amount);
        total_inr += flt(row.amount_inr);
    });

    frm.set_value("custom_total_transaction_currency", total_transaction_currency);
    frm.set_value("custom_total_inr", total_inr);
}



20) custom allowance by company calculate
Disabled


frappe.ui.form.on('Quotation', {
    refresh(frm) {
        calculate_total_cost_by_customer(frm);
    },

    // Recalculate total when table is modified
    custom_allowance1_add: function(frm) {
        calculate_total_cost_by_customer(frm);
    },
    custom_allowance1_remove: function(frm) {
        calculate_total_cost_by_customer(frm);
    }
});

frappe.ui.form.on('Costing Detail', {
    days__journeys: function(frm, cdt, cdn) {
        calculate_row_total_and_parent_total(frm, cdt, cdn);
    },
    person: function(frm, cdt, cdn) {
        calculate_row_total_and_parent_total(frm, cdt, cdn);
    },
    cost_per_unit: function(frm, cdt, cdn) {
        calculate_row_total_and_parent_total(frm, cdt, cdn);
    }
});

function calculate_row_total_and_parent_total(frm, cdt, cdn) {
    let row = locals[cdt][cdn];
    row.total_cost = (row.days__journeys || 0) * (row.person || 0) * (row.cost_per_unit || 0);
    frm.refresh_field("custom_allowance1");
    calculate_total_cost_by_customer(frm);
}

function calculate_total_cost_by_customer(frm) {
    let total = 0;
    (frm.doc.custom_allowance1 || []).forEach(row => {
        total += row.total_cost || 0;
    });
    frm.set_value('custom_total_cost_by_company', total);
}

