# Salesforce Custom Get API to dynamically Transform Data

Apex Trigger :

```
@RestResource(urlMapping = '/getLead/*')
global class GetLeadsAPI {
    
    @HttpGet
    global static ResponseWrapper getLead(){
        ResponseWrapper responseWrapObj = new ResponseWrapper();
        try{
            RestRequest request = RestContext.request;
            
            String query = request.params.get('query');
            String transformers = request.params.get('transformers');
            
            Map<String,Object> transformersMap = new Map<String,Object>();
            
            if(transformers != null){
                transformersMap = (Map<String,Object>)JSON.deserializeUntyped(transformers);
            }

            //Map to store Field Name and Formula Instance 
            Map<String,FormulaEval.FormulaInstance> mapofAPINameToFormulaInstance = new Map<String,FormulaEval.FormulaInstance>();
            
            Set<String> fieldNames = new Set<String>();

            //Map to store Enum Values for all data types
            Map<String,FormulaEval.FormulaReturnType> mapofTypeToEnum = new Map<String,FormulaEval.FormulaReturnType>();
            mapofTypeToEnum.put('boolean',FormulaEval.FormulaReturnType.BOOLEAN);
            mapofTypeToEnum.put('date',FormulaEval.FormulaReturnType.DATE);
            mapofTypeToEnum.put('datetime',FormulaEval.FormulaReturnType.DATETIME);
            mapofTypeToEnum.put('decimal',FormulaEval.FormulaReturnType.DECIMAL);
            mapofTypeToEnum.put('id',FormulaEval.FormulaReturnType.ID);
            mapofTypeToEnum.put('integer',FormulaEval.FormulaReturnType.INTEGER);
            mapofTypeToEnum.put('long',FormulaEval.FormulaReturnType.LONG);
            mapofTypeToEnum.put('string',FormulaEval.FormulaReturnType.STRING);
            mapofTypeToEnum.put('time',FormulaEval.FormulaReturnType.TIME);
            
            for(String fieldApiName : transformersMap.keySet()){
                Transformers transformObj = (Transformers)JSON.deserialize(JSON.serialize(transformersMap.get(fieldApiName)),Transformers.class);
                if(!mapofTypeToEnum.containsKey(transformObj.dataType.toLowerCase())){  //Checking if unsupported data type is passed
                    responseWrapObj.error = 'Invalid Data Type ' + transformObj.dataType + ' on field ' + fieldApiName;
                    return responseWrapObj;
                }
                try{
                    //Creating the instance of Formula
                    FormulaEval.FormulaInstance formulaInstanceObj = Formula.builder()
                        .withType(Schema.Lead.class)
                        .withReturnType(mapofTypeToEnum.get(transformObj.dataType.toLowerCase()))
                        .withFormula(transformObj.Formula)
                        .withGlobalVariables(new List<FormulaEval.FormulaGlobal>{FormulaEval.FormulaGlobal.Label,FormulaEval.FormulaGlobal.ORGANIZATION})
                        .build();

                    //Adding the instance to Map
                    mapofAPINameToFormulaInstance.put(fieldApiName,formulaInstanceObj);
                }catch(Exception ex){
                    responseWrapObj.error = 'Error in Formula because ' + ex.getMessage();
                    return responseWrapObj;
                }
            }
            
            List<Lead> listofLead = Database.query(query);
            
            List<Map<String,Object>> listofLeadsToReturn = new List<Map<String,Object>>();
            
            for(Lead leadObj : listofLead){
                Map<String,Object> leadMap = (Map<String,Object>)JSON.deserializeUntyped(JSON.serialize(leadObj));
                if(mapofAPINameToFormulaInstance.size() > 0){
                    for(String fieldApiName : mapofAPINameToFormulaInstance.keySet()){
                        FormulaEval.FormulaInstance formulaInstanceObj = mapofAPINameToFormulaInstance.get(fieldApiName);
                        Object formulaResult;
                        try{
                            formulaResult = formulaInstanceObj.evaluate(leadObj);      //Evaluating or Calculating the Formula
                        }catch(Exception ex){
                            responseWrapObj.error = 'Runtime error on evaluating formula ' + ex.getMessage();
                            return responseWrapObj;
                        }
                        leadMap.remove('attributes');  leadMap.remove('type');     // Removing unnecessary attributes
                        leadMap.put(fieldApiName,formulaResult);
                    }
                }
                leadMap.remove('attributes');  leadMap.remove('type');
                listofLeadsToReturn.add(leadMap);
            }
            responseWrapObj.data = JSON.serialize(listofLeadsToReturn);
        }catch(Exception ex){
            responseWrapObj.error = 'Internal Server Error - ' + ex.getMessage();
        }
        return responseWrapObj;
    }
    
    
    public class Transformers{
        public String dataType;
        public String Formula;
    }
    
    global class ResponseWrapper{
        
        public String data;
        public String error;
        
    }
    
}
```

Sample CURL : 

```
curl --location --globoff 'https://resourceful-bear-e88akt-dev-ed.trailblaze.my.salesforce.com/services/apexrest/getLead?query=select%20id%2CFirstName%2CLastName%2CNumberofLocations__c%2CFirst_Disbursement_Amount__c%2CSubsequent_Disbursement_Amount__c%20from%20lead&transformers={%22Full%20Name%22%20%3A%20{%22dataType%22%20%3A%20%22integer%22%2C%20%22Formula%22%20%3A%22(First_Disbursement_Amount__c%2BSubsequent_Disbursement_Amount__c)*VALUE(%24Label.Interest_Rate)%22}}' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--header 'Authorization: Bearer XXXXXXXXXXXXXXXXXXXXXXXXXXXXX' \
--header 'Cookie: BrowserId=xelvDM0hEfCq1dF-7tI3TQ; CookieConsentPolicy=0:1; LSKey-c$CookieConsentPolicy=0:1'
```

