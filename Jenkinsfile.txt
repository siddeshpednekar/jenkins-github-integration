#!groovy
import groovy.json.JsonSlurper

node {
    stage('Read and Execute tests') {
        // Obtain credentials for accessing the useMango server, which should be stored in Jenkins with the ID of 'usemango'
        withCredentials([
                string(credentialsId: 'useMangoApiKey', variable: 'useMangoApiKey')
        ]) {
            // Obtain values for the server url, project name and folder name from a Jenkins config file, which have the ID
            // that matches the name of the current pipeline job.
            String TEST_SERVICE_URL = "https://tests.api.usemango.co.uk/v1"
            String SCRIPTS_SERVICE_URL = "https://scripts.api.usemango.co.uk/v1"
            String APP_WEBSITE_URL = "https://app.usemango.co.uk"
            echo "Running tests in project ${params['Project ID']} with tags ${params['Tags']}"
            def (envId, envName) = getEnvIdAndName(TEST_SERVICE_URL)
            def tests = getTests(TEST_SERVICE_URL)
            def testJobs = [:]
            def testResults = [:]
            Integer count = 0
            tests.eachWithIndex { test, index ->
                echo "Scheduling ${test.Name}"
                testJobs[test.Id] = {
                    node('usemango') {
                        wrap([$class: "MaskPasswordsBuildWrapper", varPasswordPairs: [[password: '%useMangoApiKey%']]]) {
                            dir ("${env.WORKSPACE}\\${tests[index].Id}") {
                                deleteDir()
                            }
                            dir("${env.WORKSPACE}\\${tests[index].Id}") {
                                List<Map> scenarioList = tests[index].Scenarios
                                String datasetType = "";
                                Map paramMap = [:]
                                boolean isMultiDataset = false;
                                if (scenarioList != null) {
                                    if (scenarioList.size() == 1) {
                                        datasetType = "Default Dataset";
                                    } else {
                                        isMultiDataset = true
                                        datasetType = "Multi Dataset 'Dataset Count=${scenarioList.size()}'"
                                    }
                                    paramMap["scenario"] = scenarioList.collect { it.Id}
                                }
                                paramMap["environment"] = envId
                                String url = addQueryParameterToUrl(SCRIPTS_SERVICE_URL + "/tests/" + tests[index].Id.toString(), paramMap).toString()
                                bat "curl -s --create-dirs -L -D \"response.txt\" -X GET \"${url}\" -H \"Authorization: APIKEY " + '%useMangoApiKey%' +"\" --output \"${tests[index].Id}.pyz\""
                                String httpCode = powershell(returnStdout: true, script: "Write-Output (Get-Content \"response.txt\" | select -First 1 | Select-String -Pattern '.*HTTP/1.1 ([^\\\"]*) *').Matches.Groups[1].Value")
                                echo "Test executable response code - ${httpCode}"
                                if (httpCode.contains("200")) {
                                    echo "Executing - '${tests[index].Name}' ${datasetType}"
                                    try {
                                        bat "\"%UM_PYTHON_PATH%\" ${tests[index].Id}.pyz -k " + '%useMangoApiKey%' + " -j result.xml"
                                        String run_id = getRunId()
                                        if (run_id != null) {
                                            testResults[count] = "TestName: '${tests[index].Name}' ${datasetType} (Passed) - ${APP_WEBSITE_URL}/p/${params['Project ID']}/executions/${run_id}"
                                        } else {
                                            testResults[count] = "TestName: '${tests[index].Name}' ${datasetType} (Failed) - ${isMultiDataset ? 'multidataset_run.log' : 'run.log' } not generated"
                                        }
                                    } catch(Exception ex) {
                                        String run_id = getRunId()
                                        testResults[count] = "TestName: '${tests[index].Name}' ${datasetType} (Failed) - Exception occured: ${ex.getMessage()} - ${APP_WEBSITE_URL}/p/${params['Project ID']}/executions/${run_id}"
                                    } finally{
                                        if (fileExists("result.xml")){
                                            junit "result.xml"
                                        } else {
                                            echo "Test failed to generate JUNIT file"
                                        }
                                    }
                                } else {
                                    testResults[count] = "TestName: '${tests[index].Name}' ${datasetType} (Failed) - Unable to get scripted test: ${httpCode}"
                                }
                                count++
                            }
                        }
                    }
                }
            }
            parallel testJobs
            boolean allPassed = true
            int passed = 0
            int failed = 0
            echo "useMango Execution on '${envName}' environment, results: "
            testResults.eachWithIndex { result, index ->
                echo "${index + 1}. ${result.value}"
                if (result.value.contains("Failed")){
                    allPassed = false
                    failed += 1
                }
                else {
                    passed += 1
                }
            }
            String testsExecutedMsg = "Total Tests: ${testResults.size()}"
            if (isRunWithDatasetOptionSelected()) {
                testsExecutedMsg += " NOTE: This represents the number of tests run. The consolidated report for each test contains information about the datasets executed."
            }
            echo testsExecutedMsg
            echo "Passed: ${passed}"
            echo "Failed: ${failed}"
            if (!allPassed){
                error("Not all the tests passed.")
            }
        }
    }
}

def getRunId() {
    if (fileExists("multidataset_run.log")) {
        return powershell(returnStdout: true, script: 'Write-Output (Get-Content .\\multidataset_run.log | select -First 1 | Select-String -Pattern \'.*\\"RunId\\": \\"([^\\"]*)\\"\').Matches.Groups[1].Value')
    }
    if (fileExists("run.log")) {
        return powershell(returnStdout: true, script: 'Write-Output (Get-Content .\\run.log | select -First 1 | Select-String -Pattern \'.*\\"RunId\\": \\"([^\\"]*)\\"\').Matches.Groups[1].Value')
    }
    return null
}

// Read all tests with the tags specified
def getTests(String baseUrl) {
    String cursor = ""
    def tests = []
    def jsonSlurper = new JsonSlurper()
    echo "Retrieved the following tests from project ${params['Project ID']} with the tags ${params['Tags']} and status ${params['Status']}"
    while(true) {
        URL url = new URL("${baseUrl}/projects/${params['Project ID']}/testindex?tags=${params['Tags']}&status=${params['Status']}&cursor=${cursor}")
        def testPage = getRequest(url, "Testindex")
        testPage.Items.each{ test ->
            def scenarios = getScenarios(baseUrl, test.Id)
            tests << [Id: test.Id, Name: test.Name, Scenarios: scenarios]
        }
        if (!testPage.Info.HasNext){
            break
        }
        cursor = testPage.Info.Next
    }
    return tests
}

def getScenarios(String baseUrl, String testId){
    def scenarios = [[Id: "0", Name: "Default"]]
    def runWithDataset = isRunWithDatasetOptionSelected()
    if (runWithDataset) {
        def isScenarioPresentForTest = scenariosPresent(baseUrl, testId)
        if (isScenarioPresentForTest) {
            URL url = new URL("${baseUrl}/projects/${params['Project ID']}/tests/${testId}/scenarios")
            def scenarioPage = getRequest(url, "Scenarios")
            if (scenarioPage != null) {
                scenarioPage.each { scenario ->
                    scenarios << [Id: scenario.Id, Name: scenario.Name]
                }
            }
            return scenarios;
        }
    }
    return null
}

def addQueryParameterToUrl(String path, Map<String, Object> queryParams) {
    if (queryParams.isEmpty()) {
        return new URL(path)
    }
    path = path + "?"
    for (final def keyValue in queryParams.entrySet()) {
        def query = keyValue.getKey()
        def queryValue = keyValue.getValue()
        if (queryValue instanceof ArrayList) {
            queryValue.each { value ->
                path += "${query}=${URLEncoder.encode(value as String, 'UTF-8')}&"
            }
        } else {
            path += "${query}=${URLEncoder.encode(queryValue as String, "UTF-8")}&"
        }
    }
    return new URL(path.substring(0, path.length() - 1))
}

boolean scenariosPresent(String baseUrl, String testId) {
    URL url = new URL("${baseUrl}/projects/${params['Project ID']}/tests/${testId}")
    def scenarioPage = getRequest(url, "Dataset");
    return scenarioPage["Parameters"].size() >= 1
}

boolean isRunWithDatasetOptionSelected() {
    def value = params['Run with datasets']
    if (value != null) {
        return value
    }
    // backward compatibility
    value = params['Run with scenarios']
    if (value != null) {
        return value
    }
}

def getEnvIdAndName(String baseUrl) {
    echo "Retrieving environment"
    if (params['Environment'] == null || params['Environment'].toString().isEmpty()) {
        def (envId, envName) = getDefaultEnv(baseUrl);
        echo "Using the current default environment '${envName}'"
        return [envId, envName];
    }
    def env = getEnvByName(baseUrl, params['Environment'].toString());
    if (env == null) {
        throw new NoSuchElementException("The '${params['Environment']}' environment does not exists")
    }
    return env;
}

def getDefaultEnv(String baseUrl) {
    URL url = new URL("${baseUrl}/projects/${params['Project ID']}/environments/default");
    def environment = getRequest(url, "Default Environment");
    return [environment["Id"], environment["Name"]]
}

def getEnvByName(String baseUrl, String envName) {
    String cursor = ""
    boolean fetchEnvironmentPage = true
    while (fetchEnvironmentPage) {
        URL url = addQueryParameterToUrl("${baseUrl}/projects/${params['Project ID']}/environments",  [
                q: envName,
                cursor: cursor
        ])
        def environments = getRequest(url, "Environment");
        for (environment in environments["Items"]) {
            if (environment["Name"].toString() == envName) {
                return [environment["Id"], environment["Name"]]
            }
        }
        cursor = environments.Info.Next
        fetchEnvironmentPage = environments.Info.HasNext
    }
    return null;
}

def getRequest(URL url, String requestedFor) {
    HttpURLConnection conn = url.openConnection()
    conn.setRequestMethod("GET")
    conn.setDoInput(true)
    conn.setRequestProperty("Authorization", "APIKEY $useMangoApiKey")
    conn.connect()
    if (conn.responseCode == 200) {
        String content = conn.getInputStream().getText()
        return new JsonSlurper().parseText(content)
    }
    throw new Exception("${requestedFor} GET request failed with code: ${conn.responseCode}")
}
