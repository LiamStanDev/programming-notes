
# String and Regex
### string_view
* 他是一個胖指針
* 只讀
* 兼容各種字符串類型的自動轉換，作為參數很方便
* 切片語法: `{&mystr[1],4}`，會建立出一個 `string_view`


### regex
學習怎麼操作:
* `reg ex_match()`: Match a regular expression against a string (of known size).
* `reg ex_search()`: Search for a string that matches a regular expression in an (arbitrarily long) stream of data.
* `reg ex_replace()`: Search for strings that match a regular expression in an (arbitrarily long) stream of data and replace them. 
* `reg ex_iterator`: Iterate over matches and submatches
* `reg ex_token_iterator`: Iterate over non-matches.

