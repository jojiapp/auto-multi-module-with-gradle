# [Gradle] 멀티 모듈 자동화

멀티 모듈을 어떻게 구성할 것인가는 권용근님의 우아한세미나 영상이 너무 좋기 때문에 해당 영상을 참고하였지만
멀티 모듈을 어떻게 `Gradle`로 만드는지에 대한 자세한 내용은 잘 없었습니다.

그렇게 멀티 모듈 관련해서 찾아보다가 유튜브에 `옥탑방 개발자`라는 분의 멀티 모듈 영상을 봤는데
`settings.gradle`을 이용하여 셋팅하는 것을 보고 너무 좋은것 같아 따라하며 제 입맛에 맞게 조금 수정하여 보았습니다.

그럼 시작해 보겠습니다.

## settings.gradle

보통 `settings.gradle`에는 `rootProject.name`와 `include`정도만 적어 두지만
저는 자동으로 생성하도록 구성하였습니다.

여기서 저는 모듈을 감싸는 폴더들을 감안하여 폴더의 가장 안쪽에 위치해 있는 것만 모듈로 만들도록
재귀를 통해 구현하였습니다.

### 전체 로직

```groovy
rootProject.name = 'multi-module-test'

def modules = [
        "domain",
        "web",
        "server"
]

modules.forEach(module -> {

    def moduleDir = file(rootDir, "${rootProject.name}-${module}")
    if (!moduleDir.exists()) {
        moduleDir.mkdirs()
    }

    makeModules(moduleDir)

})

private void makeModules(final File moduleDir, final String parentDirNames = "") {

    for (final def subModule in moduleDir.listFiles()) {
        if (subModule.file) {
            continue
        }

        if (isModuleDir(subModule)) {
            makeModules(subModule, "${parentDirNames}:${moduleDir.name}")
            continue
        }

        makeBuildGradleFile(subModule)
        makeSrcDir(subModule)
        includeModule(parentDirNames, moduleDir, subModule)

    }
}

private void makeBuildGradleFile(final File subModule) {

    def subModuleDir = file(subModule, "build.gradle")
    if (!subModuleDir.exists()) {
        subModuleDir.text = getDefaultGradleSetting()
    }
}

private static String getDefaultGradleSetting() {
    return """dependencies {

}

bootJar { enabled = false }
jar { enabled = true }

"""
}

private void makeSrcDir(subModule) {

    ["src/main/java/com/jojiapp/multimoduletest",
     "src/main/resources",
     "src/test/java/com/jojiapp/multimoduletest",
     "src/test/resources"
    ].forEach(src -> {
        def srcDir = file(subModule, src)
        if (!srcDir.exists()) {
            srcDir.mkdirs()
        }
    })
}

private File file(final File dir, final String name) {

    return file("${dir.absolutePath}/${name}")
}

private void includeModule(
        final String parentDirNames,
        final File moduleDir,
        final File subModule
) {

    def projectName = "${parentDirNames}:${moduleDir.name}:${subModule.name}"
    include projectName
    project(projectName).projectDir = subModule
}

private static boolean isModuleDir(final File subModule) {

    return subModule.listFiles().size() != 0 && !subModule.list().contains("src")
}
```

대부분 폴더를 생성하는 로직이라 특별한 것은 없습니다.

### 폴더를 모듈로 인식시키기

```groovy
private void includeModule(
        final String parentDirNames,
        final File moduleDir,
        final File subModule
) {

    def projectName = "${parentDirNames}:${moduleDir.name}:${subModule.name}"
    include projectName
    project(projectName).projectDir = subModule
}

private static boolean isModuleDir(final File subModule) {

    return subModule.listFiles().size() != 0 && !subModule.list().contains("src")
}
```

위 로직을 통해 해당 폴더가 모듈로써 인식이 됩니다.

이렇게 하면 뎁스 구조로 모듈을 볼 수 있어 좋습니다

> 뎁스 구조가 아닌 그냥 각각을 모듈로써 사용하고 샆다면 `:`대신 `-`를 사용하는 등 구분자를 바꿔주면 됩니다.
>
> `:`는 모듈의 상하관계를 나타냅니다.

### src 파일 생성

```groovy
private void makeSrcDir(subModule) {

    ["src/main/java/com/jojiapp/multimoduletest",
     "src/main/resources",
     "src/test/java/com/jojiapp/multimoduletest",
     "src/test/resources"
    ].forEach(src -> {
        def srcDir = file(subModule, src)
        if (!srcDir.exists()) {
            srcDir.mkdirs()
        }
    })
}
```

위 로직을 통해 기본적으로 `src` 내에 필요한 폴더를 생성하여 줍니다.

> 하지만 이렇게 한다고 소스 파일로 인식되지 않습니다.
>
> 소스 파일로써 인식 시킬려면 `build.gradle`에서 `java-library`플러그인을 추가해야 합니다.

## build.gradle

`build.gradle`에서 전체적인 모듈에 대한 설정을 할 수 있습니다.

### 전체 로직

```groovy
buildscript {

    ext {
        spring = "org.springframework"
        boot = "${spring}.boot"
        bootVersion = "2.7.4"
    }

    repositories {
        mavenCentral()
    }

    dependencies {
        classpath "$boot:spring-boot-gradle-plugin:$bootVersion"
    }
}

allprojects {

    apply plugin: "java-library"
    apply plugin: boot
    apply plugin: "io.spring.dependency-management"

    group "com.jojiapp"
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '17'
}

subprojects {

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    repositories {
        mavenCentral()
    }

    tasks.named('test') {
        useJUnitPlatform()
    }
}
```

- `buildscript`: 태스크를 실행할 때 사용되는 값입니다.
- `allprojects`: 전체 모듈에 사용될 공통 설정
- `subprojects`: `rootProject`를 제외한 모듈에서 사용할 공통 설정

```groovy
apply plugin: "java-library"
```

위 모듈들이 사용하도록 설정하여야 아까 만든 `src` 폴더를 소스 파일로 인식합니다.

이제 `root`에서 폴더를 만들어 `gradle`만 실행시키면 모듈이 만들어 집니다.

> 플러그인으로 `java`를 사용하면 `dependencies`를 추가할 때 `api()`구문을 사용할 수 없기 때문에
>
> `java-library`를 사용하는 것을 권장합니다.

## 참고 사이트

- [https://www.youtube.com/watch?v=1ZiBjduthSg&t=1553s](https://www.youtube.com/watch?v=1ZiBjduthSg&t=1553s)
- [https://www.youtube.com/watch?v=nH382BcycHc&t=4196s](https://www.youtube.com/watch?v=nH382BcycHc&t=4196s)
