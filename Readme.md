**Run CI/CD**

```
docker run -d --user root  -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home  -v /var/run/docker.sock:/var/run/docker.sock bbmokhtari/jenkins-docker
```

Copy password from docker image `jenkins_home`

```
cp [CONTAINER ID]:/var/jenkins_home/secrets/initialAdminPassword .
```

Add settings to build.gradle

```

ext.versionMajor = 0
ext.versionMinor = 1
```
```
defaultConfig{
        testBuildType System.getProperty('testBuildType', 'debug')
        }
```
```
versionName computeVersionName()
versionCode computeVersionCode()
```
```
def computeVersionName() {
    // Basic <major>.<minor> version name
    return String.format('%d.%d', versionMajor, versionMinor)
}

// Will use Jenkins build number as well
def computeVersionCode() {
    return (versionMajor * 100000) + (versionMinor * 10000) + Integer.valueOf(System.env.BUILD_NUMBER ?: 0)
}

task printVersion {
    doLast {
        print android.defaultConfig.versionName + '.' + android.defaultConfig.versionCode
    }
}
```
