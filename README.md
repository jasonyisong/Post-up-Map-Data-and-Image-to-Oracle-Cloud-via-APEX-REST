# Post-up-Map-Data-and-Image-to-Oracle-Cloud-via-APEX-REST

Upload Local Table Data to Cloud DB by using Oracle APEX.

Created by Yi Song 2022.08.19


# How to post up the map data and images from local Oracle database tables to Oracle Cloud DB and easily view and share location information and venue images on maps in the cloud.

Easy solution with Oracle APEX services in Oracle CLoud.

## 1. Create the same table structure in Oracle Cloud Database as the local tables.

```
CREATE TABLE  "MAP_DATA" 
   (	"ID" NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  NOT NULL ENABLE, 
	"NAME" VARCHAR2(100) COLLATE "USING_NLS_COMP", 
	"CITY" VARCHAR2(100) COLLATE "USING_NLS_COMP", 
	"REGION" VARCHAR2(100) COLLATE "USING_NLS_COMP", 
	"LOCATION" VARCHAR2(250) COLLATE "USING_NLS_COMP", 
	"VENUE" VARCHAR2(100) COLLATE "USING_NLS_COMP", 
	"LATITUDE" NUMBER, 
	"LONGITUDE" NUMBER, 
	 PRIMARY KEY ("ID")
  USING INDEX  ENABLE
   )  DEFAULT COLLATION "USING_NLS_COMP"
/

CREATE TABLE  "VENUE" 
   (	"ID" NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  NOT NULL ENABLE, 
	"NAME" VARCHAR2(100) COLLATE "USING_NLS_COMP" NOT NULL ENABLE, 
	"IMAGE" BLOB, 
	"IMAGE_NAME" VARCHAR2(250) COLLATE "USING_NLS_COMP", 
	"MIMETYPE" VARCHAR2(50) COLLATE "USING_NLS_COMP", 
	"NOTES" VARCHAR2(500) COLLATE "USING_NLS_COMP", 
	 PRIMARY KEY ("ID")
  USING INDEX  ENABLE
   )  DEFAULT COLLATION "USING_NLS_COMP"
/
```
Notes: This data structure is for demonstration purposes only. If used as a production system, it is best to create a foreign key in the MAP_DATA table to associate with the ID of the VENUE table.

## 2. Create Module / Resource / Handler by using APEX RESTful Data Services in Oracle Cloud

#### 2.1 The first URI is MAP_DATA 

Method: POST \
Source Type: PL/SQL \
Mime Types Allowed: application/json

Source:

```
DECLARE
    -- The post json data content
    l_body CLOB := :body_text;
BEGIN
    -- Init status
    :output := 0;
    
    -- If you have the key, you can post the data
    IF :key = 'your key' THEN
    
    	-- Delete all records first
        DELETE FROM map_data;

	-- Insert MAP_DATA table from json clob
        INSERT INTO map_data (
            name,
            city,
            region,
            location,
            venue,
            latitude,
            longitude
        )
            SELECT
                name,
                city,
                region,
                location,
                venue,
                latitude,
                longitude
            FROM
                JSON_TABLE ( l_body, '$[*]'
                    COLUMNS (
                        name,
                        city,
                        region,
                        location,
                        venue,
                        latitude,
                        longitude
                    )
                );

        COMMIT;
	
	-- If insert successful, output status
	:output := 100;
    END IF;
    
EXCEPTION
    WHEN OTHERS THEN
        -- If error, output status 
        :output := 400;
END;
```

Parameters:

|Name  |Bind Variable|Access Method|Source Type|Data Type|Comments|
|------|-------------|-------------|-----------|---------|--------|
|key   |key          |IN           |HTTP HEADER|STRING   |        |
|output|output       |OUT          |RESPONSE   |INTEGER  |        |


#### 2.2 The second URI is VENUE 

Method: POST \
Source Type: PL/SQL \
Mime Types Allowed: application/json

Source:
```
declare
    -- The post json data content
    l_body clob := :body_text;
begin
    -- Init status
    :output := 0;
    
    -- If you have the key, you can post the data
    if :key='your key' then
    
    -- Delete all records first
    DELETE FROM venue;
    
    -- Insert VENUE table from json clob
    INSERT INTO venue (
        name,
        image,
        image_name,
        mimetype,
        notes
    )
        SELECT
            name,
	    -- Convert image clobbase64 to blob
            apex_web_service.clobbase642blob(image) image, 
            image_name,
            mimetype,
            notes
        FROM
            json_table ( 
	    l_body, '$[*]'
	  columns ( 
		name,
		-- image is clob data type
		image clob,
		image_name,
		mimetype,
		notes
	  )
	);
	commit;
	-- If insert successful, output status
	:output := 100;
    end if;
 
EXCEPTION
    WHEN OTHERS THEN
    	-- If error, output status
        :output := 400;
END;
```

Parameters:

|Name  |Bind Variable|Access Method|Source Type|Data Type|Comments|
|------|-------------|-------------|-----------|---------|--------|
|key   |key          |IN           |HTTP HEADER|STRING   |        |
|output|output       |OUT          |RESPONSE  |INTEGER  |        |

## 3. Write the PL/SQL process in the local APEX page so that the user can click the submit button to upload the data.

Upload MAP_DATA process

```
DECLARE
    l_clob      CLOB;
    l_body_clob CLOB;
BEGIN
    apex_web_service.g_request_headers.DELETE();
    apex_web_service.g_request_headers(1).name := 'content-type';
    apex_web_service.g_request_headers(1).value := 'application/json';
    apex_web_service.g_request_headers(2).name := 'key';
    apex_web_service.g_request_headers(2).value := 'your key';
    
    -- Query all data to json format
    SELECT
        JSON_ARRAYAGG(
            JSON_OBJECT(
                name,
                city,
                region,
                location,
                venue,
                latitude,
                longitude
            )
        RETURNING CLOB)
    INTO l_body_clob
    FROM
        map_data;

    -- Post the json content to Oracle Cloud
    l_clob := apex_web_service.make_rest_request(
            p_url         => 'your oracle cloud full url',
	    p_http_method => 'post',
	    p_body        => l_body_clob
    );

    -- Get the returned status code
    IF apex_web_service.g_status_code = 100 THEN
        :P_MSG:='SUCCESS';
    ELSE
        :P_MSG:='ERROR:' || apex_web_service.g_status_code;
    END IF;

END;
```

Upload VENUE process

```
DECLARE
    l_clob      CLOB;
    l_body_clob CLOB;
BEGIN
    apex_web_service.g_request_headers.DELETE();
    apex_web_service.g_request_headers(1).name := 'content-type';
    apex_web_service.g_request_headers(1).value := 'application/json';
    apex_web_service.g_request_headers(2).name := 'key';
    apex_web_service.g_request_headers(2).value := 'your key';
    
    -- Query all data to json format
    SELECT
        JSON_ARRAYAGG(
            JSON_OBJECT(
                name,
		-- Covert blob image to clobbase64
                'image' value apex_web_service.blob2clobbase64(image) returning clob,
                image_name,
                mimetype,
                notes
            )
        RETURNING CLOB)
    INTO l_body_clob
    FROM
        venue;

    -- Post the json content to Oracle Cloud
    l_clob := apex_web_service.make_rest_request(
    	    p_url         => 'your oracle cloud full url',
	    p_http_method => 'post',
	    p_body        => l_body_clob
    );
    
    -- Get the returned status code
    IF apex_web_service.g_status_code = 100 THEN
        :P_MSG:='SUCCESS';
    ELSE
        :P_MSG:='ERROR:' || apex_web_service.g_status_code;
    END IF;
    
END;

```

## 4. Then we can easily make a map application by using the Oracle APEX service in Oracle Cloud. The map is shown below

![image](https://user-images.githubusercontent.com/33503189/185690041-0801b04d-3672-4796-ad62-bc244e5204ad.png)



Enjoy!  :)


