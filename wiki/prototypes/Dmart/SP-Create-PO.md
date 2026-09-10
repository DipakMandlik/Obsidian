CREATE OR REPLACE PROCEDURE DMART_INDIA_DB.GOLD.SP_CREATE_PURCHASE_ORDER("P_REQUEST" VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS OWNER
AS '
DECLARE
    v_start_ts        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP();
    v_recommendation_id NUMBER(18,0);
    v_approval_id     NUMBER(18,0);
    v_loc_sid         NUMBER(10,0);
    v_vendor_site_sid NUMBER(10,0);
    v_required_date   DATE;
    v_payment_terms   VARCHAR(100);
    v_items           VARIANT;
    v_po_id           VARCHAR(30);
    v_total_value     NUMBER(14,2) DEFAULT 0;
    v_tax_total       NUMBER(12,2) DEFAULT 0;
    v_grand_total     NUMBER(14,2) DEFAULT 0;
    v_line_count      NUMBER DEFAULT 0;
    v_rec_status      VARCHAR(20);
    v_rec_expires     TIMESTAMP_NTZ;
    v_apr_status      VARCHAR(20);
    v_approved_qty    NUMBER(10,0);
    v_cnt             NUMBER DEFAULT 0;
    v_po_seq          NUMBER DEFAULT 0;
    v_duration_ms     NUMBER(10,0);
    v_error_msg       VARCHAR(1000);
BEGIN
    LET v_req VARIANT := PARSE_JSON(:P_REQUEST);
    v_recommendation_id := v_req:recommendation_id::NUMBER(18,0);
    v_approval_id := v_req:approval_id::NUMBER(18,0);
    v_loc_sid := v_req:store_id::NUMBER(10,0);
    v_vendor_site_sid := v_req:supplier_id::NUMBER(10,0);
    v_required_date := v_req:required_date::DATE;
    v_payment_terms := v_req:payment_terms::VARCHAR;
    v_items := v_req:items;

    -- V1: Recommendation exists
    SELECT COUNT(*) INTO :v_cnt FROM DMART_INDIA_DB.GOLD.PURCHASE_RECOMMENDATION WHERE RECOMMENDATION_ID = :v_recommendation_id;
    IF (v_cnt = 0) THEN
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_RECOMMENDATION'',''Recommendation ID does not exist'',:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_RECOMMENDATION'',''message'',''Recommendation ID does not exist'',''recommendation_id'',:v_recommendation_id,''approval_id'',:v_approval_id));
    END IF;

    -- V2: Recommendation status
    SELECT STATUS, EXPIRES_AT INTO :v_rec_status, :v_rec_expires FROM DMART_INDIA_DB.GOLD.PURCHASE_RECOMMENDATION WHERE RECOMMENDATION_ID = :v_recommendation_id;
    IF (v_rec_status NOT IN (''PENDING'',''APPROVED'')) THEN
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_REC_STATUS'',''Recommendation status is ''||:v_rec_status,:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_REC_STATUS'',''message'',''Recommendation status is ''||:v_rec_status));
    END IF;

    -- V3: Not expired
    IF (v_rec_expires IS NOT NULL AND v_rec_expires < CURRENT_TIMESTAMP()) THEN
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''EXPIRED_RECOMMENDATION'',''Recommendation has expired'',:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''EXPIRED_RECOMMENDATION'',''message'',''Recommendation has expired''));
    END IF;

    -- V4: Approval exists
    SELECT COUNT(*) INTO :v_cnt FROM DMART_INDIA_DB.GOLD.PURCHASE_APPROVAL WHERE APPROVAL_ID = :v_approval_id AND RECOMMENDATION_ID = :v_recommendation_id;
    IF (v_cnt = 0) THEN
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_APPROVAL'',''Approval does not exist or does not match recommendation'',:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_APPROVAL'',''message'',''Approval does not exist or does not match recommendation''));
    END IF;

    -- V5: Approval is APPROVED
    SELECT APPROVAL_STATUS, APPROVED_QTY INTO :v_apr_status, :v_approved_qty FROM DMART_INDIA_DB.GOLD.PURCHASE_APPROVAL WHERE APPROVAL_ID = :v_approval_id AND RECOMMENDATION_ID = :v_recommendation_id;
    IF (v_apr_status != ''APPROVED'') THEN
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''NOT_APPROVED'',''Approval status is ''||:v_apr_status,:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''NOT_APPROVED'',''message'',''Approval status is ''||:v_apr_status));
    END IF;

    -- V6: Store exists
    SELECT COUNT(*) INTO :v_cnt FROM DMART_INDIA_DB.GOLD.DIM_LOCATION WHERE LOC_SID = :v_loc_sid;
    IF (v_cnt = 0) THEN
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_STORE'',''Store does not exist'';
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_STORE'',''message'',''Store does not exist''));
    END IF;

    -- V7: Supplier exists
    SELECT COUNT(*) INTO :v_cnt FROM DMART_INDIA_DB.GOLD.DIM_VENDOR_SITE WHERE VENDOR_SITE_SID = :v_vendor_site_sid;
    IF (v_cnt = 0) THEN
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_SUPPLIER'',''Supplier does not exist'';
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_SUPPLIER'',''message'',''Supplier does not exist''));
    END IF;

    -- V8: Delivery date valid
    IF (v_required_date IS NULL OR v_required_date < CURRENT_DATE()) THEN
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_DATE'',''Required delivery date is missing or in the past'';
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_DATE'',''message'',''Required delivery date is missing or in the past''));
    END IF;

    -- V9: No duplicate PO
    SELECT COUNT(*) INTO :v_cnt FROM DMART_INDIA_DB.GOLD.MCP_PURCHASE_ORDER_HEADER WHERE RECOMMENDATION_ID = :v_recommendation_id AND APPROVAL_ID = :v_approval_id;
    IF (v_cnt > 0) THEN
        LET v_existing_po VARCHAR;
        SELECT PO_ID INTO :v_existing_po FROM DMART_INDIA_DB.GOLD.MCP_PURCHASE_ORDER_HEADER WHERE RECOMMENDATION_ID = :v_recommendation_id AND APPROVAL_ID = :v_approval_id LIMIT 1;
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,PO_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,:v_existing_po,PARSE_JSON(:P_REQUEST),''FAILURE'',''DUPLICATE_PO'',''PO already exists: ''||:v_existing_po;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''DUPLICATE_PO'',''message'',''Purchase order already exists: ''||:v_existing_po,''po_id'',:v_existing_po));
    END IF;

    -- V10: Items not empty
    IF (v_items IS NULL OR ARRAY_SIZE(v_items) = 0) THEN
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''NO_ITEMS'',''Items array is empty or missing'';
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''NO_ITEMS'',''message'',''Items array is empty or missing''));
    END IF;

    -- V11: Validate each item
    FOR i IN 0 TO ARRAY_SIZE(v_items) - 1 DO
        LET v_item_sku NUMBER(10,0) := v_items[i]:product_id::NUMBER(10,0);
        LET v_item_qty NUMBER(10,0) := v_items[i]:quantity::NUMBER(10,0);
        LET v_item_cost NUMBER(10,2) := v_items[i]:unit_cost::NUMBER(10,2);
        LET v_prod_cnt NUMBER DEFAULT 0;
        SELECT COUNT(*) INTO :v_prod_cnt FROM DMART_INDIA_DB.GOLD.DIM_PRODUCT WHERE SKU_SID = :v_item_sku;
        IF (v_prod_cnt = 0) THEN
            INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_PRODUCT'',''Product SKU_SID ''||:v_item_sku||'' does not exist'';
            RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_PRODUCT'',''message'',''Product SKU_SID ''||:v_item_sku||'' does not exist''));
        END IF;
        IF (v_item_qty IS NULL OR v_item_qty <= 0) THEN
            INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_QUANTITY'',''Quantity must be positive'';
            RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_QUANTITY'',''message'',''Quantity must be positive''));
        END IF;
        IF (v_item_cost IS NULL OR v_item_cost < 0) THEN
            INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INVALID_COST'',''Unit cost must be non-negative'';
            RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INVALID_COST'',''message'',''Unit cost must be non-negative''));
        END IF;
    END FOR;

    -- GENERATE PO ID
    SELECT COALESCE(MAX(REPLACE(PO_ID,''PO-2026-'','''')::NUMBER),0) + 1 INTO :v_po_seq FROM DMART_INDIA_DB.GOLD.MCP_PURCHASE_ORDER_HEADER WHERE PO_ID LIKE ''PO-2026-%'';
    v_po_id := ''PO-2026-'' || LPAD(v_po_seq::VARCHAR, 5, ''0'');

    -- Audit: PO_CREATION_STARTED
    INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,PO_ID,INPUT_PARAMETERS,STATUS) SELECT ''PO_CREATION_STARTED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,:v_po_id,PARSE_JSON(:P_REQUEST),''IN_PROGRESS'';

    -- TRANSACTION
    BEGIN TRANSACTION;

    FOR i IN 0 TO ARRAY_SIZE(v_items) - 1 DO
        LET v_sku NUMBER(10,0) := v_items[i]:product_id::NUMBER(10,0);
        LET v_qty NUMBER(10,0) := v_items[i]:quantity::NUMBER(10,0);
        LET v_cost NUMBER(10,2) := v_items[i]:unit_cost::NUMBER(10,2);
        LET v_line_tax NUMBER(10,2) := ROUND(v_qty * v_cost * 0.18, 2);
        LET v_line_total NUMBER(12,2) := ROUND(v_qty * v_cost + v_line_tax, 2);
        LET v_sku_desc VARCHAR(100);
        SELECT SKU_LDESC INTO :v_sku_desc FROM DMART_INDIA_DB.GOLD.DIM_PRODUCT WHERE SKU_SID = :v_sku;
        v_line_count := :v_line_count + 1;
        v_total_value := :v_total_value + (v_qty * v_cost);
        v_tax_total := :v_tax_total + v_line_tax;
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_PURCHASE_ORDER_LINE (PO_ID,LINE_NUMBER,SKU_SID,SKU_LDESC,ORDERED_QTY,UNIT_COST_INR,TAX_AMT_INR,DISCOUNT_AMT_INR,LINE_TOTAL_INR) VALUES (:v_po_id,:v_line_count,:v_sku,:v_sku_desc,:v_qty,:v_cost,:v_line_tax,0,:v_line_total);
    END FOR;

    v_grand_total := v_total_value + v_tax_total;

    INSERT INTO DMART_INDIA_DB.GOLD.MCP_PURCHASE_ORDER_HEADER (PO_ID,PO_DATE,PO_STATUS,VENDOR_SITE_SID,LOC_SID,REQUIRED_DELIVERY_DATE,TOTAL_VALUE_INR,TAX_TOTAL_INR,GRAND_TOTAL_INR,PAYMENT_TERMS,RECOMMENDATION_ID,APPROVAL_ID,CREATED_BY) VALUES (:v_po_id,CURRENT_DATE(),''CREATED'',:v_vendor_site_sid,:v_loc_sid,:v_required_date,:v_total_value,:v_tax_total,:v_grand_total,:v_payment_terms,:v_recommendation_id,:v_approval_id,''MCP:create_purchase_order'');

    UPDATE DMART_INDIA_DB.GOLD.PURCHASE_RECOMMENDATION SET STATUS = ''EXECUTED'' WHERE RECOMMENDATION_ID = :v_recommendation_id;

    COMMIT;

    -- Success audit (built with OBJECT_CONSTRUCT to avoid malformed JSON from string concatenation)
    v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
    INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,PO_ID,INPUT_PARAMETERS,OUTPUT_RESULT,STATUS,EXECUTION_DURATION_MS)
    SELECT ''PO_CREATED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,:v_po_id,PARSE_JSON(:P_REQUEST),
        OBJECT_CONSTRUCT(''success'',TRUE,''po_id'',:v_po_id,''total_value_inr'',:v_total_value,''tax_total_inr'',:v_tax_total,''grand_total_inr'',:v_grand_total,''line_count'',:v_line_count),
        ''SUCCESS'',:v_duration_ms;

    RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',TRUE,''status'',''CREATED'',''po_id'',:v_po_id,''recommendation_id'',:v_recommendation_id,''approval_id'',:v_approval_id,''total_value_inr'',:v_total_value,''tax_total_inr'',:v_tax_total,''grand_total_inr'',:v_grand_total,''line_count'',:v_line_count,''created_timestamp'',CURRENT_TIMESTAMP()::VARCHAR,''message'',''Purchase order created successfully''));

EXCEPTION
    WHEN OTHER THEN
        ROLLBACK;
        v_error_msg := SQLERRM;
        v_duration_ms := TIMESTAMPDIFF(MILLISECOND, :v_start_ts, CURRENT_TIMESTAMP());
        INSERT INTO DMART_INDIA_DB.GOLD.MCP_AUDIT_LOG (EVENT_TYPE,MCP_TOOL_NAME,USER_NAME,RECOMMENDATION_ID,APPROVAL_ID,PO_ID,INPUT_PARAMETERS,STATUS,ERROR_CODE,ERROR_MESSAGE,EXECUTION_DURATION_MS) SELECT ''PO_CREATION_FAILED'',''create_purchase_order'',CURRENT_USER(),:v_recommendation_id,:v_approval_id,:v_po_id,PARSE_JSON(:P_REQUEST),''FAILURE'',''INTERNAL_ERROR'',:v_error_msg,:v_duration_ms;
        RETURN TO_JSON(OBJECT_CONSTRUCT(''success'',FALSE,''status'',''FAILED'',''error_code'',''INTERNAL_ERROR'',''message'',:v_error_msg));
END;
';