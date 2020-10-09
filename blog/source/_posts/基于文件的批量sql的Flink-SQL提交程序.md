---
title: 基于文件的批量sql的Flink SQL提交程序
date: 2020-10-09 22:14:09
tags:
categories:
	- Flink
---




### 背景

将sql写入文件中，可以写入多条sql，支持写入set、create、insert。然后程序读取文件提交运行程序。

### Flink版本
1.11.2 


### 代码

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.SqlParserException;
import org.apache.flink.table.api.StatementSet;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;
import java.util.concurrent.ExecutionException;

public class SQLSubmit {

    private String sqlFilePath;
    private StreamTableEnvironment tableEnvironment;
    private StatementSet statementSet;


    public static void main(String[] args) throws Exception {
        final CliOptions options = CliOptionsParser.parseClient(args);
        SQLSubmit submit = new SQLSubmit(options);
        submit.run();
    }

    private SQLSubmit(CliOptions options) {
        this.sqlFilePath = options.getSqlFilePath();
    }


    public void run() throws IOException, ExecutionException, InterruptedException {
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance()
                .inStreamingMode()
                .useBlinkPlanner()
                .build();
        this.tableEnvironment = StreamTableEnvironment.create(streamEnv, settings);
        statementSet = tableEnvironment.createStatementSet();
        List<String> sql = Files.readAllLines(Paths.get(sqlFilePath));
        List<SqlCommandParser.SqlCommandCall> calls = SqlCommandParser.parse(sql);
        for (SqlCommandParser.SqlCommandCall call : calls) {
            callCommand(call);
        }
        statementSet.execute();

    }

    // --------------------------------------------------------------------------------------------

    private void callCommand(SqlCommandParser.SqlCommandCall cmdCall) {
        switch (cmdCall.command) {
            case SET:
                callSet(cmdCall);
                break;
            case CREATE_TABLE:
                callCreateTable(cmdCall);
                break;
            case INSERT_INTO:
                callInsertInto(cmdCall);
                break;
            default:
                throw new RuntimeException("Unsupported command: " + cmdCall.command);
        }
    }

    private void callSet(SqlCommandParser.SqlCommandCall cmdCall) {
        String key = cmdCall.operands[0];
        String value = cmdCall.operands[1];
        tableEnvironment.getConfig().getConfiguration().setString(key, value);
    }

    private void callCreateTable(SqlCommandParser.SqlCommandCall cmdCall) {
        String ddl = cmdCall.operands[0];
        try {
            tableEnvironment.executeSql(ddl);
        } catch (SqlParserException e) {
            throw new RuntimeException("SQL parse failed:\n" + ddl + "\n", e);
        }
    }

    private void callInsertInto(SqlCommandParser.SqlCommandCall cmdCall) {
        String dml = cmdCall.operands[0];
        try {
            statementSet.addInsertSql(dml);
        } catch (SqlParserException e) {
            throw new RuntimeException("SQL parse failed:\n" + dml + "\n", e);
        }
    }


}

```

```java
public class CliOptions {
    private final String sqlFilePath;

    public CliOptions(String sqlFilePath) {
        this.sqlFilePath = sqlFilePath;
    }

    public String getSqlFilePath() {
        return sqlFilePath;
    }



}

```
```java
import org.apache.commons.cli.*;

public class CliOptionsParser {


    public static final Option OPTION_SQL_FILE = Option
            .builder("f")
            .required(true)
            .longOpt("file")
            .numberOfArgs(1)
            .argName("SQL file path")
            .desc("The SQL file path.")
            .build();

    public static final Options CLIENT_OPTIONS = getClientOptions(new Options());

    public static Options getClientOptions(Options options) {
        options.addOption(OPTION_SQL_FILE);
        return options;
    }

    // --------------------------------------------------------------------------------------------
    //  Line Parsing
    // --------------------------------------------------------------------------------------------

    public static CliOptions parseClient(String[] args) {
        if (args.length < 1) {
            throw new RuntimeException("./sql-submit  -f <sql-file>");
        }
        try {
            DefaultParser parser = new DefaultParser();
            CommandLine line = parser.parse(CLIENT_OPTIONS, args, true);
            return new CliOptions(
                    line.getOptionValue(CliOptionsParser.OPTION_SQL_FILE.getOpt())
            );
        } catch (ParseException e) {
            throw new RuntimeException(e.getMessage());
        }
    }


}

```
```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.function.Function;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
public class SqlCommandParser {

    private SqlCommandParser() {
        // private
    }

    public static List<SqlCommandCall> parse(List<String> lines) {
        List<SqlCommandCall> calls = new ArrayList<>();
        StringBuilder stmt = new StringBuilder();
        for (String line : lines) {
            if (line.trim().isEmpty() || line.startsWith("--")) {
                // skip empty line and comment line
                continue;
            }
            stmt.append("\n").append(line);
            if (line.trim().endsWith(";")) {
                Optional<SqlCommandCall> optionalCall = parse(stmt.toString());
                if (optionalCall.isPresent()) {
                    calls.add(optionalCall.get());
                } else {
                    throw new RuntimeException("Unsupported command '" + stmt.toString() + "'");
                }
                // clear string builder
                stmt.setLength(0);
            }
        }
        return calls;
    }

    public static Optional<SqlCommandCall> parse(String stmt) {
        // normalize
        stmt = stmt.trim();
        // remove ';' at the end
        if (stmt.endsWith(";")) {
            stmt = stmt.substring(0, stmt.length() - 1).trim();
        }

        // parse
        for (SqlCommand cmd : SqlCommand.values()) {
            final Matcher matcher = cmd.pattern.matcher(stmt);
            if (matcher.matches()) {
                final String[] groups = new String[matcher.groupCount()];
                for (int i = 0; i < groups.length; i++) {
                    groups[i] = matcher.group(i + 1);
                }
                return cmd.operandConverter.apply(groups)
                        .map((operands) -> new SqlCommandCall(cmd, operands));
            }
        }
        return Optional.empty();
    }

    // --------------------------------------------------------------------------------------------

    private static final Function<String[], Optional<String[]>> NO_OPERANDS =
            (operands) -> Optional.of(new String[0]);

    private static final Function<String[], Optional<String[]>> SINGLE_OPERAND =
            (operands) -> Optional.of(new String[]{operands[0]});

    private static final int DEFAULT_PATTERN_FLAGS = Pattern.CASE_INSENSITIVE | Pattern.DOTALL;

    /**
     * Supported SQL commands.
     */
    public enum SqlCommand {
        INSERT_INTO(
                "(INSERT\\s+INTO.*)",
                SINGLE_OPERAND),

        CREATE_TABLE(
                "(CREATE\\s+TABLE.*)",
                SINGLE_OPERAND),

        SET(
                "SET(\\s+(\\S+)\\s*=(.*))?", // whitespace is only ignored on the left side of '='
                (operands) -> {
                    if (operands.length < 3) {
                        return Optional.empty();
                    } else if (operands[0] == null) {
                        return Optional.of(new String[0]);
                    }
                    return Optional.of(new String[]{operands[1], operands[2]});
                });

        public final Pattern pattern;
        public final Function<String[], Optional<String[]>> operandConverter;

        SqlCommand(String matchingRegex, Function<String[], Optional<String[]>> operandConverter) {
            this.pattern = Pattern.compile(matchingRegex, DEFAULT_PATTERN_FLAGS);
            this.operandConverter = operandConverter;
        }

        @Override
        public String toString() {
            return super.toString().replace('_', ' ');
        }

        public boolean hasOperands() {
            return operandConverter != NO_OPERANDS;
        }
    }

    /**
     * Call of SQL command with operands and command type.
     */
    public static class SqlCommandCall {
        public final SqlCommand command;
        public final String[] operands;

        public SqlCommandCall(SqlCommand command, String[] operands) {
            this.command = command;
            this.operands = operands;
        }

        public SqlCommandCall(SqlCommand command) {
            this(command, new String[0]);
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (o == null || getClass() != o.getClass()) {
                return false;
            }
            SqlCommandCall that = (SqlCommandCall) o;
            return command == that.command && Arrays.equals(operands, that.operands);
        }

        @Override
        public int hashCode() {
            int result = Objects.hash(command);
            result = 31 * result + Arrays.hashCode(operands);
            return result;
        }

        @Override
        public String toString() {
            return command + "(" + Arrays.toString(operands) + ")";
        }
    }


}

```


### Reference



https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/create.html

http://wuchong.me/blog/2019/09/02/flink-sql-1-9-read-from-kafka-write-into-mysql/