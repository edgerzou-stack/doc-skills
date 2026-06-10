# Verilog 提取脚本分析与修改建议（按 GPT-5.4 分析）

> 说明：仓库中未找到截图对应的 Perl 文件，本分析基于你截图中的代码内容逐行推断，给出可直接套用的修改方案。

## 1. 这个脚本现在在做什么

从截图看，这个脚本的目标是：

1. 读取一个 Verilog 文件。
2. 按 `module ... endmodule` 把每个模块切出来。
3. 对每个模块提取三类信息并打印成 Tcl 字典：
   - `ports`：模块头部端口列表
   - `regs`：`reg` 声明出来的寄存器名
   - `bbox`：子模块例化信息（设计名 + 实例名）
4. 最终输出形如：

```tcl
set modules [dict create \
  modA {{p0 p1} {r0 r1} {{u0 subA} {u1 subB}}} \
  modB {{...} {...} {...}} \
]
```

## 2. 你问的 `endmodule` 结尾问题

你截图里的循环条件是：

```perl
while ($line !~ /endmodule\s*$/) {
    $line .= <$fh>;
    redo unless eof();
}
```

结论：

- `endmodule` 后面只有空格：可以匹配（`\s*$` 已覆盖）。
- `endmodule : xxx`：**不能匹配**（因为 `:` 和后续字符不在 `\s*` 内）。
- `endmodule // 注释`：通常也可能失败（取决于注释是否先被清理）。

建议改成“兼容标签和行尾注释”的模式：

```perl
my $endmodule_pattern = qr/^\s*endmodule\b(?:\s*:\s*[A-Za-z_][A-Za-z0-9_$]*)?\s*(?://.*)?$/m;

while ($line !~ /$endmodule_pattern/) {
    my $next = <$fh>;
    last unless defined $next;
    $next =~ s/\/\/[^\n]*//g;
    $line .= $next;
}
```

## 3. 增加 signal list 排除提取（推荐实现）

### 3.1 增加第二个参数：排除名单文件

```perl
if (@ARGV < 1 || @ARGV > 2) {
    die "usage: $0 <verilog_file> [exclude_signal_list]\n";
}

my $verilog_file = $ARGV[0];
my $exclude_file = $ARGV[1];
my %skip_signals;

if (defined $exclude_file && $exclude_file ne "") {
    open(my $efh, '<', $exclude_file) or die "cannot open $exclude_file:$!\n";
    while (my $sig = <$efh>) {
        chomp $sig;
        $sig =~ s/#.*$//;          # 支持 # 注释
        $sig =~ s{//.*$}{};        # 支持 // 注释
        $sig =~ s/^\s+|\s+$//g;
        next if $sig eq "";
        $sig =~ s/\[[^\]]+\]$//;   # 统一去掉末尾位选，如 data[3]
        $skip_signals{$sig} = 1;
    }
    close($efh);
}

sub normalize_sig {
    my ($name) = @_;
    $name //= "";
    $name =~ s/^\s+|\s+$//g;
    $name =~ s/\[[^\]]+\]$//;
    return $name;
}

sub should_skip_sig {
    my ($name) = @_;
    my $k = normalize_sig($name);
    return ($k ne "" && exists $skip_signals{$k});
}
```

### 3.2 在 `extract_ports` 里过滤

在 `push @port_lists, $port;` 前加：

```perl
$port = normalize_sig($port);
next if should_skip_sig($port);
push @port_lists, $port;
```

### 3.3 在 `extract_regs` 里过滤

把 `push @reg_lists, $2;` 改为：

```perl
my $reg_name = normalize_sig($2);
next if should_skip_sig($reg_name);
push @reg_lists, $reg_name;
```

## 4. 一个容易忽略的小风险（顺手建议）

你截图中的模块名正则类似：

```perl
my $module_pattern = qr/^\s*module\s+([A-Za-z0-9][A-Za-z0-9_\$]*)/;
```

首字符允许数字，这不符合 Verilog 标识符规则。建议改成：

```perl
my $module_pattern = qr/^\s*module\s+([A-Za-z_][A-Za-z0-9_\$]*)/;
```

## 5. 使用示例

`exclude_signals.list`：

```text
clk
rst_n
debug_mode
data_tmp[0]
```

运行：

```bash
perl your_extract_script.pl input.v exclude_signals.list > modules_dict.tcl
```

---

如果你把截图里的 Perl 原文件贴到仓库（比如 `extract_modules.pl`），我可以下一步直接给你打成完整补丁版本，避免你手工拼接。
