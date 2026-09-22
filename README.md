1. Database Setup (Shared Schema)
Create Token Table:

SQL
CREATE TABLE app_sso_tokens (
    token_id      VARCHAR2(100) PRIMARY KEY,
    username      VARCHAR2(100) NOT NULL,
    app_id        NUMBER        NOT NULL,
    expiry_date   TIMESTAMP     NOT NULL,
    created_at    TIMESTAMP DEFAULT SYSTIMESTAMP
);
Create SSO Package:

SQL
CREATE OR REPLACE PACKAGE pkg_apex_sso AS
    FUNCTION generate_token (p_username IN VARCHAR2, p_app_id IN NUMBER) RETURN VARCHAR2;
    FUNCTION validate_token (p_token IN VARCHAR2, p_app_id IN NUMBER) RETURN VARCHAR2;
END pkg_apex_sso;
/

CREATE OR REPLACE PACKAGE BODY pkg_apex_sso AS
    FUNCTION generate_token (p_username IN VARCHAR2, p_app_id IN NUMBER) RETURN VARCHAR2 IS
        v_token VARCHAR2(100) := RAWTOHEX(SYS_GUID());
    BEGIN
        INSERT INTO app_sso_tokens (token_id, username, app_id, expiry_date)
        VALUES (v_token, p_username, p_app_id, SYSTIMESTAMP + INTERVAL '2' MINUTE);
        COMMIT;
        RETURN v_token;
    END generate_token;

    FUNCTION validate_token (p_token IN VARCHAR2, p_app_id IN NUMBER) RETURN VARCHAR2 IS
        v_username VARCHAR2(100);
    BEGIN
        SELECT username INTO v_username FROM app_sso_tokens
        WHERE token_id = p_token AND app_id = p_app_id AND expiry_date > SYSTIMESTAMP;
        
        DELETE FROM app_sso_tokens WHERE token_id = p_token;
        COMMIT;
        RETURN v_username;
    EXCEPTION WHEN NO_DATA_FOUND THEN RETURN NULL;
    END validate_token;
END pkg_apex_sso;
/
2. Application 1 (Source App) Implementation
Button/Process Code: Generate the token and store it in an item (:P1_SSO_TOKEN).

SQL
:P1_SSO_TOKEN := pkg_apex_sso.generate_token(p_username => :APP_USER, p_app_id => 200);
Branch: Redirect user to App 2 login URL with the token parameter:
.../login?p9999_sso_token=&P1_SSO_TOKEN.

3. Application 2 (Target App) Implementation
Login Page Process (Before Header): Catch the token, validate it, and create a session programmatically.

SQL
DECLARE
    v_username VARCHAR2(100);
BEGIN
    IF :P9999_SSO_TOKEN IS NOT NULL THEN
        v_username := pkg_apex_sso.validate_token(p_token => :P9999_SSO_TOKEN, p_app_id => :APP_ID);
        IF v_username IS NOT NULL THEN
            apex_custom_auth.define_user_session(p_user => v_username, p_session_id => APEX_CUSTOM_AUTH.GET_NEXT_SESSION_ID);
            apex_util.redirect_url('f?p=' || :APP_ID || ':1:' || :APP_SESSION);
        END IF;
    END IF;
END;
