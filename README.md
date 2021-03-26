## Java Annotation Processing and Creating a Builder

# 1. Introdução
Este artigo é uma introdução ao processamento de anotação de nível de origem Java e fornece exemplos de uso dessa técnica para gerar arquivos de origem adicionais durante a compilação.

# 2. Aplicações de processamento de anotações
O processamento de anotação em nível de origem apareceu pela primeira vez em Java 5. É uma técnica útil para gerar arquivos de origem adicionais durante o estágio de compilação.

Os arquivos de origem não precisam ser arquivos Java - você pode gerar qualquer tipo de descrição, metadados, documentação, recursos ou qualquer outro tipo de arquivo, com base em anotações em seu código-fonte.

O processamento de anotação é usado ativamente em muitas bibliotecas Java onipresentes, por exemplo, para gerar metaclasses em QueryDSL e JPA, para aumentar as classes com código clichê na biblioteca Lombok.

Uma coisa importante a se notar é a limitação da API de processamento de anotações - ela só pode ser usada para gerar novos arquivos, não para alterar os existentes.

A exceção notável é a biblioteca do Lombok, que usa o processamento de anotação como um mecanismo de inicialização para se incluir no processo de compilação e modificar o AST por meio de algumas APIs do compilador interno. Essa técnica de hacky não tem nada a ver com o propósito pretendido de processamento de anotação e, portanto, não é discutida neste artigo.

# 3. API de processamento de anotações
O processamento da anotação é feito em várias rodadas. Cada rodada começa com o compilador procurando as anotações nos arquivos de origem e escolhendo os processadores de anotação adequados para essas anotações. Cada processador de anotação, por sua vez, é chamado nas fontes correspondentes.

Se algum arquivo for gerado durante este processo, outra rodada é iniciada com os arquivos gerados como sua entrada. Este processo continua até que nenhum novo arquivo seja gerado durante o estágio de processamento.

Cada processador de anotação, por sua vez, é chamado nas fontes correspondentes. Se algum arquivo for gerado durante este processo, outra rodada é iniciada com os arquivos gerados como sua entrada. Esse processo continua até que nenhum novo arquivo seja gerado durante o estágio de processamento.

A API de processamento de anotações está localizada no pacote javax.annotation.processing. A interface principal que você terá que implementar é a interface Processor, que possui uma implementação parcial na forma da classe AbstractProcessor. Esta classe é a que iremos estender para criar nosso próprio processador de anotação.

# 4. Configurando o Projeto
Para demonstrar as possibilidades de processamento de anotação, desenvolveremos um processador simples para gerar construtores de objetos fluentes para classes anotadas.

Vamos dividir nosso projeto em dois módulos Maven. Um deles, o módulo processador de anotações, conterá o próprio processador junto com a anotação, e outro, o módulo usuário de anotação, conterá a classe anotada. Este é um caso de uso típico de processamento de anotação.

As configurações do módulo processador de anotações são as seguintes. Vamos usar a biblioteca de autoatendimento do Google para gerar o arquivo de metadados do processador, que será discutido mais tarde, e o plugin maven-compiler ajustado para o código-fonte Java 8. As versões dessas dependências são extraídas para a seção de propriedades.

As versões mais recentes da biblioteca de auto-serviço e maven-compiler-plugin podem ser encontradas no repositório Maven Central:

```
<properties>
    <auto-service.version>1.0-rc2</auto-service.version>
    <maven-compiler-plugin.version>
      3.5.1
    </maven-compiler-plugin.version>
</properties>

<dependencies>

    <dependency>
        <groupId>com.google.auto.service</groupId>
        <artifactId>auto-service</artifactId>
        <version>${auto-service.version}</version>
        <scope>provided</scope>
    </dependency>

</dependencies>

<build>
    <plugins>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>${maven-compiler-plugin.version}</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>

    </plugins>
</build>
```

O módulo Maven do usuário de anotação com as fontes anotadas não precisa de nenhum ajuste especial, exceto adicionar uma dependência no módulo do processador de anotação na seção de dependências:

```
<dependency>
    <groupId>com.isaccanedo</groupId>
    <artifactId>annotation-processing</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

# 5. Definindo uma anotação
Suponha que temos uma classe POJO simples em nosso módulo de usuário de anotação com vários campos:

```
public class Person {

    private int age;

    private String name;

    // getters and setters …

}
```

Queremos criar uma classe auxiliar builder para instanciar a classe Person com mais fluência:

```
Person person = new PersonBuilder()
  .setAge(25)
  .setName("John")
  .build();
```

Esta classe PersonBuilder é uma escolha óbvia para uma geração, pois sua estrutura é completamente definida pelos métodos configuradores de Person.

Vamos criar uma anotação @BuilderProperty no módulo processador de anotações para os métodos setter. Isso nos permitirá gerar a classe Builder para cada classe que tem seus métodos setter anotados:

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface BuilderProperty {
}
```

A anotação @Target com o parâmetro ElementType.METHOD garante que essa anotação só possa ser colocada em um método.

A política de retenção SOURCE significa que essa anotação está disponível apenas durante o processamento de origem e não está disponível no tempo de execução.

A classe Person com propriedades anotadas com a anotação @BuilderProperty terá a seguinte aparência:

```
public class Person {

    private int age;

    private String name;

    @BuilderProperty
    public void setAge(int age) {
        this.age = age;
    }

    @BuilderProperty
    public void setName(String name) {
        this.name = name;
    }

    // getters …

}
```

# 6. Implementando um processador
### 6.1. Criação de uma subclasse AbstractProcessor
Começaremos estendendo a classe AbstractProcessor dentro do módulo Maven do processador de anotações.

Primeiro, devemos especificar as anotações que este processador é capaz de processar e também a versão do código-fonte suportada. Isso pode ser feito implementando os métodos getSupportedAnnotationTypes e getSupportedSourceVersion da interface do Processor ou anotando sua classe com as anotações @SupportedAnnotationTypes e @SupportedSourceVersion.

A anotação @AutoService faz parte da biblioteca de autoatendimento e permite gerar os metadados do processador que serão explicados nas seções a seguir.

```
@SupportedAnnotationTypes(
  "com.isaccanedo.annotation.processor.BuilderProperty")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@AutoService(Processor.class)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, 
      RoundEnvironment roundEnv) {
        return false;
    }
}
```

Você pode especificar não apenas os nomes das classes de anotação concretas, mas também curingas, como “com.isaccanedo.annotation.*" Para processar anotações dentro do pacote com.isaccanedo.annotation e todos os seus subpacotes, ou mesmo "*" para processar todas as anotações .

O único método que teremos que implementar é o método de processo que faz o próprio processamento. É chamado pelo compilador para cada arquivo de origem que contém as anotações correspondentes.

As anotações são passadas como o primeiro Conjunto <? estende o argumento TypeElement> annotations e as informações sobre a rodada de processamento atual são passadas como o argumento RoundEnviroment roundEnv.

O valor booleano de retorno deve ser verdadeiro se o seu processador de anotação processou todas as anotações passadas e você não deseja que elas sejam passadas para outros processadores de anotação na lista.

### 6.2. Juntando informação
Nosso processador ainda não faz nada de útil, então vamos preenchê-lo com código.

Primeiro, precisaremos iterar por todos os tipos de anotação encontrados na classe - em nosso caso, o conjunto de anotações terá um único elemento correspondente à anotação @BuilderProperty, mesmo se essa anotação ocorrer várias vezes no arquivo de origem.

Ainda assim, é melhor implementar o método de processo como um ciclo de iteração, para fins de integridade:

```
@Override
public boolean process(Set<? extends TypeElement> annotations, 
  RoundEnvironment roundEnv) {

    for (TypeElement annotation : annotations) {
        Set<? extends Element> annotatedElements 
          = roundEnv.getElementsAnnotatedWith(annotation);
        
        // …
    }

    return true;
}
```

Neste código, usamos a instância RoundEnvironment para receber todos os elementos anotados com a anotação @BuilderProperty. No caso da classe Person, esses elementos correspondem aos métodos setName e setAge.

O usuário da anotação @BuilderProperty pode anotar erroneamente métodos que não são realmente configuradores. O nome do método setter deve começar com set e o método deve receber um único argumento. Então, vamos separar o joio do trigo.

No código a seguir, usamos o coletor Collectors.partitioningBy() para dividir os métodos anotados em duas coleções: setters anotados corretamente e outros métodos anotados erroneamente:

```
Map<Boolean, List<Element>> annotatedMethods = annotatedElements.stream().collect(
  Collectors.partitioningBy(element ->
    ((ExecutableType) element.asType()).getParameterTypes().size() == 1
    && element.getSimpleName().toString().startsWith("set")));

List<Element> setters = annotatedMethods.get(true);
List<Element> otherMethods = annotatedMethods.get(false);
```

Aqui, usamos o método Element.asType() para receber uma instância da classe TypeMirror que nos dá alguma capacidade de introspectar tipos, embora estejamos apenas no estágio de processamento de origem.

Devemos alertar o usuário sobre métodos anotados incorretamente, então vamos usar a instância Messager acessível a partir do campo protegido AbstractProcessor.processingEnv. As linhas a seguir gerarão um erro para cada elemento anotado erroneamente durante o estágio de processamento de origem:

```
otherMethods.forEach(element ->
  processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
    "@BuilderProperty must be applied to a setXxx method " 
      + "with a single argument", element));
```

Obviamente, se a coleção de setters correta estiver vazia, não há nenhum ponto de continuar a iteração de conjunto de elemento de tipo atual:

```
if (setters.isEmpty()) {
    continue;
}
```

Se a coleção setters tiver pelo menos um elemento, vamos usá-lo para obter o nome da classe totalmente qualificado do elemento envolvente, que no caso do método setter parece ser a própria classe de origem:

```
String className = ((TypeElement) setters.get(0)
  .getEnclosingElement()).getQualifiedName().toString();
```

A última informação de que precisamos para gerar uma classe builder é um mapa entre os nomes dos setters e os nomes de seus tipos de argumento:


```
Map<String, String> setterMap = setters.stream().collect(Collectors.toMap(
    setter -> setter.getSimpleName().toString(),
    setter -> ((ExecutableType) setter.asType())
      .getParameterTypes().get(0).toString()
));
```

### 6.3. Gerando o arquivo de saída
Agora temos todas as informações de que precisamos para gerar uma classe de construtor: o nome da classe de origem, todos os seus nomes de setter e seus tipos de argumento.

Para gerar o arquivo de saída, usaremos a instância Filer fornecida novamente pelo objeto na propriedade protegida AbstractProcessor.processingEnv:

```
JavaFileObject builderFile = processingEnv.getFiler()
  .createSourceFile(builderClassName);
try (PrintWriter out = new PrintWriter(builderFile.openWriter())) {
    // writing generated file to out …
}
```

O código completo do método writeBuilderFile é fornecido abaixo. Precisamos apenas calcular o nome do pacote, o nome da classe do construtor totalmente qualificado e os nomes das classes simples para a classe de origem e a classe do construtor. O resto do código é bastante direto.

```
private void writeBuilderFile(
  String className, Map<String, String> setterMap) 
  throws IOException {

    String packageName = null;
    int lastDot = className.lastIndexOf('.');
    if (lastDot > 0) {
        packageName = className.substring(0, lastDot);
    }

    String simpleClassName = className.substring(lastDot + 1);
    String builderClassName = className + "Builder";
    String builderSimpleClassName = builderClassName
      .substring(lastDot + 1);

    JavaFileObject builderFile = processingEnv.getFiler()
      .createSourceFile(builderClassName);
    
    try (PrintWriter out = new PrintWriter(builderFile.openWriter())) {

        if (packageName != null) {
            out.print("package ");
            out.print(packageName);
            out.println(";");
            out.println();
        }

        out.print("public class ");
        out.print(builderSimpleClassName);
        out.println(" {");
        out.println();

        out.print("    private ");
        out.print(simpleClassName);
        out.print(" object = new ");
        out.print(simpleClassName);
        out.println("();");
        out.println();

        out.print("    public ");
        out.print(simpleClassName);
        out.println(" build() {");
        out.println("        return object;");
        out.println("    }");
        out.println();

        setterMap.entrySet().forEach(setter -> {
            String methodName = setter.getKey();
            String argumentType = setter.getValue();

            out.print("    public ");
            out.print(builderSimpleClassName);
            out.print(" ");
            out.print(methodName);

            out.print("(");

            out.print(argumentType);
            out.println(" value) {");
            out.print("        object.");
            out.print(methodName);
            out.println("(value);");
            out.println("        return this;");
            out.println("    }");
            out.println();
        });

        out.println("}");
    }
}
```

# 7. Executando o Exemplo
Para ver a geração do código em ação, você deve compilar ambos os módulos da raiz pai comum ou primeiro compilar o módulo do processador de anotações e, em seguida, o módulo do usuário de anotações.

A classe PersonBuilder gerada pode ser encontrada dentro do arquivo 
annotation-user/target/generated-sources/annotations/com/isaccanedo/annotation/PersonBuilder.java e deve ter a seguinte aparência:

```
package com.isaccanedo.annotation;

public class PersonBuilder {

    private Person object = new Person();

    public Person build() {
        return object;
    }

    public PersonBuilder setName(java.lang.String value) {
        object.setName(value);
        return this;
    }

    public PersonBuilder setAge(int value) {
        object.setAge(value);
        return this;
    }
}
```

8. Formas alternativas de registrar um processador

Para usar seu processador de anotações durante o estágio de compilação, você tem várias outras opções, dependendo do seu caso de uso e das ferramentas que usa.

### 8.1. Usando a ferramenta do processador de anotação
A ferramenta apt era um utilitário de linha de comando especial para processar arquivos de origem. Fazia parte do Java 5, mas desde o Java 7 foi descontinuado em favor de outras opções e removido completamente no Java 8. Não será discutido neste artigo.

### 8.2. Usando a chave do compilador
A chave do compilador -processor é um recurso JDK padrão para aumentar o estágio de processamento de origem do compilador com seu próprio processador de anotação.

Observe que o próprio processador e a anotação já devem estar compilados como classes em uma compilação separada e presentes no caminho de classe, portanto, a primeira coisa que você deve fazer é:

```
javac com/isaccanedo/annotation/processor/BuilderProcessor
javac com/isaccanedo/annotation/processor/BuilderProperty
```

Em seguida, você faz a compilação real de suas fontes com a chave -processor especificando a classe do processador de anotação que acabou de compilar:

```
javac -processor com.isaccanedo.annotation.processor.MyProcessor Person.java
```

Para especificar vários processadores de anotação de uma vez, você pode separar seus nomes de classe com vírgulas, como este:

```
javac -processor package1.Processor1,package2.Processor2 SourceFile.java
```

### 8.3. Usando Maven
O maven-compiler-plugin permite especificar processadores de anotação como parte de sua configuração.

Aqui está um exemplo de adição de processador de anotação para o plug-in do compilador. Você também pode especificar o diretório para colocar as fontes geradas, usando o parâmetro de configuração generatedSourcesDirectory.

Observe que a classe BuilderProcessor já deve ser compilada, por exemplo, importada de outro jar nas dependências de compilação:

```
<build>
    <plugins>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.5.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
                <generatedSourcesDirectory>${project.build.directory}
                  /generated-sources/</generatedSourcesDirectory>
                <annotationProcessors>
                    <annotationProcessor>
                        com.isaccanedo.annotation.processor.BuilderProcessor
                    </annotationProcessor>
                </annotationProcessors>
            </configuration>
        </plugin>

    </plugins>
</build>
```

### 8.4. Adicionando um Jar do Processador ao Classpath

Em vez de especificar o processador de anotação nas opções do compilador, você pode simplesmente adicionar um jar especialmente estruturado com a classe do processador ao caminho de classe do compilador.

Para pegá-lo automaticamente, o compilador precisa saber o nome da classe do processador. Portanto, você deve especificá-lo 
no arquivo META-INF/services/javax.annotation.processing.Processor como um nome de classe totalmente qualificado do processador:

```
com.isaccanedo.annotation.processor.BuilderProcessor
```

Você também pode especificar vários processadores deste jar para serem selecionados automaticamente, separando-os com uma nova linha:

```
package1.Processor1
package2.Processor2
package3.Processor3
```

Se você usar o Maven para construir este jar e tentar colocar este arquivo diretamente no diretório 
src/main/resources/META-INF/services, encontrará o seguinte erro:

```
[ERROR] Bad service configuration file, or exception thrown while 
constructing Processor object: javax.annotation.processing.Processor: 
Provider com.isaccanedo.annotation.processor.BuilderProcessor not found
```

Isso ocorre porque o compilador tenta usar esse arquivo durante o estágio de processamento de origem do próprio módulo, 
quando o arquivo BuilderProcessor ainda não foi compilado. O arquivo deve ser colocado dentro de outro diretório de recursos 
e copiado para o diretório META-INF/services durante o estágio de cópia de recursos da construção do Maven ou 
(melhor ainda) gerado durante a construção.

A biblioteca de autoatendimento do Google, discutida na seção seguinte, permite gerar este arquivo usando uma anotação simples.

### 8.5. Usando a biblioteca de autoatendimento do Google
Para gerar o arquivo de registro automaticamente, você pode usar a anotação @AutoService da biblioteca de autoatendimento do Google, desta forma:

```
@AutoService(Processor.class)
public BuilderProcessor extends AbstractProcessor {
    // …
}
```

Essa própria anotação é processada pelo processador de anotações da biblioteca de autoatendimento. Este processador gera o 
arquivo META-INF/services/javax.annotation.processing.Processor contendo o nome da classe BuilderProcessor.

# 9. Conclusão
Neste artigo, demonstramos o processamento de anotação de nível de origem usando um exemplo de geração de uma classe Builder para um POJO. Também fornecemos várias maneiras alternativas de registrar processadores de anotação em seu projeto.