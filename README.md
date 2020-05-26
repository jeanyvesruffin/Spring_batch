# Spring_batch

## Ajouter les dependences spring batch

Dans le fichier build.gradle ajouter les lignes suivantes:

	compile group: 'org.springframework.boot', name: 'spring-boot-starter-batch', version: '2.3.0.RELEASE'


	testCompile group: 'org.springframework.batch', name: 'spring-batch-test', version: '4.2.2.RELEASE'

## Configurer l'application Spring batch

1 . Ajouter une classe de configuration � votre projet avec les imports n�cessaire, avec la declaration de 3 attributs membres de classes JobRepository, JobExplorer et JobLauncher.

	```java
	import org.springframework.batch.core.configuration.annotation.BatchConfigurer;
	import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
	import org.springframework.batch.core.explore.JobExplorer;
	import org.springframework.batch.core.launch.JobLauncher;
	import org.springframework.batch.core.repository.JobRepository;
	import org.springframework.stereotype.Component;
	import org.springframework.transaction.PlatformTransactionManager;
	@Component
	@EnableBatchProcessing
	public class BatchConfiguration implements BatchConfigurer {
		private JobRepository jobRepository;
		private JobExplorer jobExplorer;
		private JobLauncher jobLauncher;
	}
	```

* JobRepository: Conserve les m�tadonn�es sur le job par lots

* JobExplorer: Recupere les m�tadonn�es du repository

* JobLauncher: Ex�cute des jobs avec des param�tres donn�s


2 . Cabler deux attributs membres de la classe PlatformTransactionManager et DataSource

	```java
	@Autowired
	@Qualifier(value="batchTransactionManager")
	private PlatformTransactionManager batchTransactionManager;
	@Autowired
	@Qualifier(value="bqtchDataSource")
	private DataSource batchDataSource;
	```

3 . Remplir le contrat d'interface de BatchConfigurer.

	```java
	@Override
	public PlatformTransactionManager getTransactionManager() throws Exception {
		return this.batchTransactionManager;
	}
	@Override
	public JobExplorer getJobExplorer() throws Exception {
		return this.jobExplorer;
	}
	```
	
4 . Ajouter la methode de creation d'une execution de job (launcher + repository + afterPropertiesSet)

	```java
	protected JobLauncher createJobLauncher() throws Exception {
		SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
		jobLauncher.setJobRepository(jobRepository);
		jobLauncher.afterPropertiesSet();
		return jobLauncher;
	}
	protected JobRepository createJobRepository() throws Exception {
		JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
		factory.setDataSource(this.batchDataSource);
		factory.setTransactionManager(getTransactionManager());
		factory.afterPropertiesSet();
		return factory.getObject();
	}
	@PostConstruct
	public void afterPropertiesSet()throws Exception {
		this.jobRepository = createJobRepository();
		JobExplorerFactoryBean jobExplorerFactoryBean = new JobExplorerFactoryBean();
		jobExplorerFactoryBean.setDataSource(this.batchDataSource);
		jobExplorerFactoryBean.afterPropertiesSet();
		this.jobExplorer = jobExplorerFactoryBean.getObject();
		this.jobLauncher = createJobLauncher();
	}
	```
	
5 . Ajouter � votre fichier de configuration application.yml

	spring:
		application:
			name: PatientBatchLoader
				batch:
					job:
						enable: false
	...
	application:
		batch:
			inputPath: D:\WORK\WorkSpace\SandBox\Spring_batch\Spring_batch\data
			
6 . On ajoute dans le fichier ApplicationProperties les propri�t�s sp�cifiques � l'application

	```java
	@ConfigurationProperties(prefix = "application", ignoreUnknownFields = false)
	public class ApplicationProperties {
		private final Batch batch = new Batch();
		public Batch getBatch() {
			return batch;
		}
		public static class Batch{
			private String inputPath = "D:/WORK/WorkSpace/SandBox/Spring_batch/Spring_batch/data";
			public String getInputPath() {
				return this.inputPath;
			}
			public void setInputPath(String inputPath) {
				this.inputPath = inputPath;
			}
		}	
	}
	```java
		
## Ajout de l'objet base de donn�e � Spring Batch

![Batch Term](Documents/batchTerm.bmp)


1 . Ajouter la ligne suivante � votre fichier master.xml
	
	<include file="config/liquibase/changelog/01012018000000_create_spring_batch_objects.xml" relativeToChangelogFile="false"/>
	
2 . Creer votre fichier 01012018000000_create_spring_batch_objects.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd
                        http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

    <property name="now" value="now()" dbms="h2"/>
    <property name="now" value="GETDATE()" dbms="mssql"/>

    <changeSet id="01012018000001" author="system">
        <createTable tableName="BATCH_JOB_INSTANCE">
            <column name="JOB_INSTANCE_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="VERSION" type="bigint"/>
            <column name="JOB_NAME" type="VARCHAR(100)">
                <constraints nullable="false" />
            </column>
            <column name="JOB_KEY" type="VARCHAR(32)">
                <constraints nullable="false" />
            </column>
        </createTable>

        <createIndex indexName="JOB_INST_UN"
                     tableName="BATCH_JOB_INSTANCE"
                     unique="true">
            <column name="JOB_NAME" type="varchar(100)"/>
            <column name="JOB_KEY" type="varchar(32)"/>
        </createIndex>

        <createTable tableName="BATCH_JOB_EXECUTION">
            <column name="JOB_EXECUTION_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="VERSION" type="bigint"/>
            <column name="JOB_INSTANCE_ID" type="bigint">
                <constraints nullable="false" />
            </column>
            <column name="CREATE_TIME" type="timestamp">
                <constraints nullable="false" />
            </column>
            <column name="START_TIME" type="timestamp" defaultValue="null"/>
            <column name="END_TIME" type="timestamp" defaultValue="null"/>
            <column name="STATUS" type="VARCHAR(10)"/>
            <column name="EXIT_CODE" type="VARCHAR(2500)"/>
            <column name="EXIT_MESSAGE" type="VARCHAR(2500)"/>
            <column name="LAST_UPDATED" type="timestamp"/>
            <column name="JOB_CONFIGURATION_LOCATION" type="VARCHAR(2500)"/>
        </createTable>

        <addForeignKeyConstraint baseColumnNames="JOB_INSTANCE_ID"
                                 baseTableName="BATCH_JOB_EXECUTION"
                                 constraintName="JOB_INST_EXEC_FK"
                                 referencedColumnNames="JOB_INSTANCE_ID"
                                 referencedTableName="BATCH_JOB_INSTANCE"/>

        <createTable tableName="BATCH_JOB_EXECUTION_PARAMS">
            <column name="JOB_EXECUTION_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="TYPE_CD" type="VARCHAR(6)">
                <constraints nullable="false" />
            </column>
            <column name="KEY_NAME" type="VARCHAR(100)">
                <constraints nullable="false" />
            </column>
            <column name="STRING_VAL" type="VARCHAR(250)"/>
            <column name="DATE_VAL" type="timestamp" defaultValue="null"/>
            <column name="LONG_VAL" type="bigint"/>
            <column name="DOUBLE_VAL" type="double precision"/>
            <column name="IDENTIFYING" type="CHAR(1)">
                <constraints nullable="false" />
            </column>
        </createTable>

        <addForeignKeyConstraint baseColumnNames="JOB_EXECUTION_ID"
                                 baseTableName="BATCH_JOB_EXECUTION_PARAMS"
                                 constraintName="JOB_EXEC_PARAMS_FK"
                                 referencedColumnNames="JOB_EXECUTION_ID"
                                 referencedTableName="BATCH_JOB_EXECUTION"/>

        <createTable tableName="BATCH_STEP_EXECUTION">
            <column name="STEP_EXECUTION_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="VERSION" type="bigint">
                <constraints nullable="false" />
            </column>
            <column name="STEP_NAME" type="varchar(100)">
                <constraints nullable="false" />
            </column>
            <column name="JOB_EXECUTION_ID" type="bigint">
                <constraints nullable="false" />
            </column>
            <column name="START_TIME" type="timestamp">
                <constraints nullable="false" />
            </column>
            <column name="END_TIME" type="timestamp" defaultValue="null"/>
            <column name="STATUS" type="varchar(10)"/>
            <column name="COMMIT_COUNT" type="bigint"/>
            <column name="READ_COUNT" type="bigint"/>
            <column name="FILTER_COUNT" type="bigint"/>
            <column name="WRITE_COUNT" type="bigint"/>
            <column name="READ_SKIP_COUNT" type="bigint"/>
            <column name="WRITE_SKIP_COUNT" type="bigint"/>
            <column name="PROCESS_SKIP_COUNT" type="bigint"/>
            <column name="ROLLBACK_COUNT" type="bigint"/>
            <column name="EXIT_CODE" type="varchar(2500)"/>
            <column name="EXIT_MESSAGE" type="varchar(2500)"/>
            <column name="LAST_UPDATED" type="timestamp"/>
        </createTable>

        <addForeignKeyConstraint baseColumnNames="JOB_EXECUTION_ID"
                                 baseTableName="BATCH_STEP_EXECUTION"
                                 constraintName="JOB_EXEC_STEP_FK"
                                 referencedColumnNames="JOB_EXECUTION_ID"
                                 referencedTableName="BATCH_JOB_EXECUTION"/>

        <createTable tableName="BATCH_STEP_EXECUTION_CONTEXT">
            <column name="STEP_EXECUTION_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="SHORT_CONTEXT" type="varchar(2500)">
                <constraints nullable="false" />
            </column>
            <column name="SERIALIZED_CONTEXT" type="LONGVARCHAR"/>
        </createTable>

        <addForeignKeyConstraint baseColumnNames="STEP_EXECUTION_ID"
                                 baseTableName="BATCH_STEP_EXECUTION_CONTEXT"
                                 constraintName="STEP_EXEC_CTX_FK"
                                 referencedColumnNames="STEP_EXECUTION_ID"
                                 referencedTableName="BATCH_STEP_EXECUTION"/>

        <createTable tableName="BATCH_JOB_EXECUTION_CONTEXT">
            <column name="JOB_EXECUTION_ID" type="bigint" autoIncrement="${autoIncrement}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="SHORT_CONTEXT" type="varchar(2500)">
                <constraints nullable="false" />
            </column>
            <column name="SERIALIZED_CONTEXT" type="LONGVARCHAR"/>
        </createTable>

        <addForeignKeyConstraint baseColumnNames="JOB_EXECUTION_ID"
                                 baseTableName="BATCH_JOB_EXECUTION_CONTEXT"
                                 constraintName="JOB_EXEC_CTX_FK"
                                 referencedColumnNames="JOB_EXECUTION_ID"
                                 referencedTableName="BATCH_JOB_EXECUTION"/>

        <createSequence sequenceName="BATCH_STEP_EXECUTION_SEQ" />
        <createSequence sequenceName="BATCH_JOB_EXECUTION_SEQ" />
        <createSequence sequenceName="BATCH_JOB_SEQ" />

    </changeSet>
</databaseChangeLog>
```

3 . Executer votre application avec spring PatientBatchLoaderApp

4 . Controler votre base de donn�e � l'adresse url:

http://localhost:8080/console

5 . Changer les proprietes de configuration du serveur H2 tel quel:

Avant:
![Console H2 avant](Documents/consoleH2Before.bmp)

Apres:
![onsole H2 apr�s](Documents/consoleH2After.bmp)
