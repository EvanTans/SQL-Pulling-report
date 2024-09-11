# SQL-Pulling-report
-- This query retrieves detailed information on procurement contracts, 
-- purchase orders, suppliers, and budget allocations for the fiscal year 2025.

SELECT 
    -- Extract the last 11 characters of the contract reference as the registration number
    RIGHT(ctr_ref, 11) AS 'Registration Number',
    
    -- Contract title and related procurement details
    ctr_label_en AS 'Contract Title',
    sup_clean_name AS 'Provider', 
    _nindu_name_en AS 'Industry',
    _pmeth_label_en AS 'Procurement Method',
    project_label_en AS 'Program',
    
    -- Purchase order (PO) information
    ord_code_calculated AS '2025 PO ID', 
    _ord_amt AS 'Budgeted Amount',
    t_ord_fiscal_year.fiscalyear_label_en AS 'PO Fiscal Year',
    
    -- Advance payment information
    _adva_code_calculated AS 'Advance ID',
    _adva_label_en AS 'Advance Label',
    _adva_reason AS 'Advance Reason',
    _adva_amt_req AS 'Advance Amount Requested',
    
    -- Conditional logic for advance payment status
    CASE 
        WHEN adva.status_code = 'dis' THEN 'Disbursed' 
        WHEN adva.status_code = 'app' THEN 'Payment Approval in Progress' 
        WHEN adva.status_code = 'can' THEN 'Canceled' 
        WHEN adva.status_code = 'dra' THEN 'Draft' 
        WHEN adva.status_code = 'okp' THEN 'Ok-to-Pay' 
        WHEN adva.status_code = 'pip' THEN 'Payment in Progress' 
        WHEN adva.status_code IS NULL THEN 'No Advance' 
    END AS 'Advance Status',
    
    -- Date fields for when the advance was created and approved for payment
    _adva_cre_date AS 'Advance Creation Date',
    _adva_ok_to_pay_date AS 'Advance Approval Date',  
    
    -- PRC2 field, used for tracking purposes
    _adva_prc2_id AS 'PRC2 ID'

FROM 
    t_ord_order AS ord
    
    -- Join to the advances table to retrieve advance payment data
    LEFT JOIN t_ord_advanc_ AS adva 
        ON ord.ord_id = adva._ord_id
    
    -- Join to the contracts table to retrieve contract details
    LEFT JOIN t_ctr_contract AS ctr 
        ON ord._ord_doc_id_label = ctr_ref
    
    -- Join to the suppliers table to retrieve supplier details
    LEFT JOIN t_sup_supplier AS sup 
        ON sup.sup_id = ctr.sup_id
    
    -- Join to the fiscal year table to retrieve the fiscal year for the PO
    LEFT JOIN t_ord_fiscal_year 
        ON ord._fiscalyear_id = t_ord_fiscal_year.fiscalyear_id
    
    -- Join to the industry table to retrieve industry details
    LEFT JOIN t_buy_nyc_industry_ AS nindu 
        ON nindu._nindu_id = ctr._nindu_id
    
    -- Join to the project table to retrieve program or project information
    LEFT JOIN t_prj_project AS project 
        ON ctr._project_id = project.project_id
    
    -- Join to procurement methods table to retrieve procurement method details
    LEFT JOIN t_buy_procurement_methods_ AS pmeth 
        ON pmeth._pmeth_code = ctr._pmeth_code

-- Apply filters: 
WHERE 
    ctr.orga_node IN ('068')  -- Filter for a specific organization node
    AND t_ord_fiscal_year.fiscalyear_label_en = '2025'  -- Only include data for the 2025 fiscal year
    AND ord.status_code = 'act'  -- Only include active purchase orders
    AND ctype_code != 'amen'  -- Exclude certain contract types (e.g., amendments)
    AND _pmeth_label_en NOT IN ('Micropurchase')  -- Exclude micropurchases

ORDER BY 
    ord_code_calculated DESC;  -- Order results by PO ID in descending order
