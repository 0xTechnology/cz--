# cz--笑哭币

继续建设。😂

按公开说过的习惯生成中文短回复：一半以上带 😂，不预测价格，不给上币时间表，拿合影当信用就先当诈骗信号。

## 用法

```bash
python -m cz_reply "什么时候上币"
python -m cz_reply -n 5 "社区都在唱空"
python -m cz_reply --thread 4 "能合作吗"
python -m cz_reply --stats

"""分类、抽样、引用。空输入经常只回一个表情。"""

from __future__ import annotations

import random
import re
from dataclasses import dataclass, field

from .lexicon import 关键词, 单句, 表情, 话题句
from .style import 最长字数, 裁短, 补上笑哭


@dataclass
class 一条回复:
    正文: str
    话题: str
    带笑哭: bool
    引用: str = ""

    def 渲染(self) -> str:
        if not self.引用:
            return self.正文
        return f"{self.正文}\n\n> {self.引用}"


@dataclass
class 回复引擎:
    笑哭概率: float = 0.55
    随机: random.Random = field(default_factory=random.Random)

    def 归类(self, 文本: str) -> str:
        低 = 文本.lower()
        for 话题, 词组 in 关键词.items():
            if any(词 in 低 for 词 in 词组):
                return 话题
        return "默认"

    def 摘一句(self, 文本: str) -> str:
        文本 = " ".join(文本.split())
        if not 文本:
            return ""
        return 文本 if len(文本) <= 36 else 文本[:33] + "..."

    def 一条(self, 来信: str = "", 强制笑哭: bool | None = None) -> 一条回复:
        来信 = 来信.strip()
        话题 = self.归类(来信) if 来信 else "默认"
        if not 来信 or self.随机.random() < 0.22:
            正文 = self.随机.choice(单句)
        else:
            主干 = self.随机.choice(话题句.get(话题, 话题句["默认"]))
            正文 = 裁短(主干)

        带了 = 表情 in 正文
        概率 = self.笑哭概率 if 强制笑哭 is None else (1.0 if 强制笑哭 else 0.0)
        if not 带了 and self.随机.random() < 概率:
            正文 = 补上笑哭(正文, 表情)
            带了 = True
        if len(正文) > 最长字数:
            正文 = 裁短(正文)
        return 一条回复(正文=正文, 话题=话题, 带笑哭=带了, 引用=self.摘一句(来信))

    def 多条(self, 来信: str = "", 条数: int = 3) -> list[一条回复]:
        条数 = max(1, min(条数, 20))
        见过: set[str] = set()
        结果: list[一条回复] = []
        保护 = 0
        while len(结果) < 条数 and 保护 < 条数 * 8:
            保护 += 1
            项 = self.一条(来信)
            if 项.正文 in 见过:
                continue
            见过.add(项.正文)
            结果.append(项)
        while len(结果) < 条数:
            结果.append(self.一条(来信))
        return 结果

    def 连回(self, 来信: str, 轮数: int = 4) -> list[一条回复]:
        """第一句接话题，后面越来越短。"""
        轮数 = max(1, min(轮数, 8))
        结果 = [self.一条(来信)]
        for i in range(1, 轮数):
            if i >= 2 and self.随机.random() < 0.6:
                正文 = self.随机.choice(单句)
                结果.append(一条回复(正文=正文, 话题="默认", 带笑哭=表情 in 正文))
            else:
                结果.append(self.一条(来信))
        return 结果

    def 抽样占比(self, 次数: int = 200) -> float:
        命中 = sum(1 for _ in range(次数) if self.一条("继续建设").带笑哭)
        return 命中 / 次数


_链接 = re.compile(r"https?://\S+")
_账号 = re.compile(r"@\w+")


def 清洗(文本: str) -> str:
    文本 = _链接.sub("", 文本)
    文本 = _账号.sub("", 文本)
    return " ".join(文本.split())


    """排成可直接粘贴的连续回复。"""

from __future__ import annotations

from .generator import 一条回复


def 排版(条目: list[一条回复], 名字: str = "cz") -> str:
    行 = [f"@{名字}"]
    for 序号, 项 in enumerate(条目, 1):
        行.append(f"{序号}. {项.渲染()}")
        行.append("")
    行.append("继续建设。😂")
    return "\n".join(行).rstrip() + "\n"

    """自检：笑哭占比应在一半附近。"""

from __future__ import annotations

from collections import Counter

from .generator import 回复引擎


def 报告(引擎: 回复引擎, 样本: int = 500, 种子句: str = "什么时候上币") -> str:
    话题计数: Counter[str] = Counter()
    笑哭数 = 0
    长度 = []
    for _ in range(样本):
        项 = 引擎.一条(种子句)
        话题计数[项.话题] += 1
        笑哭数 += int(项.带笑哭)
        长度.append(len(项.正文))
    平均 = sum(长度) / len(长度)
    比例 = 笑哭数 / 样本
    前几 = ", ".join(f"{k}:{v}" for k, v in 话题计数.most_common(4))
    结论 = "可以" if 0.45 <= 比例 <= 0.75 else "再调"
    return (
        f"样本={样本}\n"
        f"笑哭占比={比例:.2%} 目标={引擎.笑哭概率:.0%}\n"
        f"平均字数={平均:.1f}\n"
        f"话题={前几}\n"
        f"结论={结论} 😂"
    )
