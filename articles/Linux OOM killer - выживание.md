# Linux OOM killer - выживание

Когда на вашем Linux-компьютере заканчивается память, ядро вызывает **Убийцу нехватки памяти (OOM)** для освобождения памяти. Это часто встречается на серверах, на которых запущен ряд процессов с интенсивным использованием памяти. В этом посте мы немного глубже рассмотрим, когда вызывается OOM killer, как он решает, какой процесс убивать, и можем ли мы предотвратить его от таких важных процессов, как базы данных.

### Как OOM Killer выбирает, какой процесс убить?

Ядро Linux дает оценку каждому запущенному процессу, называемому `oom_score`, которая показывает, насколько вероятно, что он будет остановлен в случае нехватки доступной памяти. Оценка пропорциональна количеству памяти, используемой процессом. Оценка - 10% процентов памяти, используемой процессом. Таким образом, максимальная оценка составляет 100% x 10 = 1000. Кроме того, если процесс выполняется как **привилегированный пользователь**, он получает **немного более низкий показатель oom\_score** по сравнению с тем же самым использованием памяти обычным пользовательский процесс. В более ранних версиях Linux (ядро v2.6.32) существовала более сложная эвристика, которая рассчитывала этот показатель.

`oom_score` процесса можно найти в каталоге `/proc`. Допустим, что идентификатор процесса (pid) вашего процесса равен 42, `cat /proc/42/oom_score` даст вам оценку процесса.

### Могу ли я гарантировать, что некоторые важные процессы не будут убиты OOM Killer?

Да! Убийца OOM проверяет `oom_score_adj`, чтобы скорректировать свой окончательный расчетный счет. Этот файл присутствует в `/proc/$pid/oom_score_adj`. Вы можете добавить в этот файл большую отрицательную оценку, чтобы гарантировать, что ваш процесс будет с меньшей вероятностью быть выбранным и завершенным OOM killer. `oom_score_adj` может варьироваться от -1000 до 1000. Если вы назначите ему -1000, он может использовать 100% памяти и все же избежать прерывания OOM killer. С другой стороны, если вы назначите ему 1000, ядро Linux будет продолжать убивать процесс, даже если оно использует минимальное количество памяти.

Давайте вернемся к нашему процессу с pid 42. Вот как вы можете изменить его `oom_score_adj`:

```console
$ sudo echo -200 > /proc/42/oom_score_adj
```

Мы должны сделать это как `root` пользователь или `sudo`, потому что Linux не позволяет обычным пользователям снижать оценку OOM. Вы можете увеличить оценку OOM как обычный пользователь без каких-либо специальных разрешений. например, `echo 100> /proc/42/oom_score_adj`

Существует также другая, менее детальная оценка, называемая «oom_adj», которая варьируется от -16 до 15. Она похожа на «oom_score_adj». Фактически, когда вы устанавливаете `oom_score_adj`, ядро автоматически уменьшает его и вычисляет `oom_adj`. `oom_adj` имеет магическое значение -17, которое указывает на то, что данный процесс никогда не должен быть убит убийцей OOM.

### Показать оценки OOM всех запущенных процессов

Этот сценарий отображает оценку OOM и скорректированную оценку OOM всех запущенных процессов в порядке убывания оценки OOM

```bash
#!/bin/bash
# Displays running processes in descending order of OOM score
printf 'PID\tOOM Score\tOOM Adj\tCommand\n'
while read -r pid comm; do [ -f /proc/$pid/oom_score ] && [ $(cat /proc/$pid/oom_score) != 0 ] && printf '%d\t%d\t\t%d\t%s\n' "$pid" "$(cat /proc/$pid/oom_score)" "$(cat /proc/$pid/oom_score_adj)" "$comm"; done < <(ps -e -o pid= -o comm=) | sort -k 2nr
```

### Проверьте, был ли какой-либо из ваших процессов убит OOM

Самый простой способ - это "grep" в ваших системных журналах. В Ubuntu: `grep -i kill /var/log/syslog`. Если процесс был убит, вы можете получить такие результаты, как `my_process вызвал oom-killer: gfp_mask = 0x201da, order = 0, oom_score_adj = 0`

### Предостережения о корректировке результатов ООМ

Помните, что ООМ является признаком более серьезной проблемы - нехватки доступной памяти. Лучший способ решить эту проблему - либо увеличить доступную память (например, улучшить аппаратное обеспечение), либо перенести некоторые программы на другие машины, либо уменьшить потребление памяти программами (например, выделить меньше памяти, где это возможно).

Слишком сильная настройка скорректированного показателя OOM приведет к гибели случайных процессов и невозможности освободить достаточно памяти.

### Рекомендации

1.   [proc](http://man7.org/linux/man-pages/man5/proc.5.html) справочная страница
2.   [https://askubuntu.com/questions/60672/how-do-i-use-oom-score-adj/](https://askubuntu.com/questions/60672/how-do-i-use-oom-score-adj/) 
3.   [Пошаговое руководство](https://linux-mm.org/OOM_Killer) о том, какая часть кода Linux называется
4.  Классический [статья LWN](https://lwn.net/Articles/317814/) (немного устаревший)
5.   [Вызов убийцы OOM вручную](https://www.lynxbee.com/how-to-invoke-oom-killer-manually-for-understanding-which-process-gets-killed-first/)

**********
[OOM killer](/tags/OOM%20killer.md)
