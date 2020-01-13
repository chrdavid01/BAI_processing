# BAI_processing
Changes BAI files into a CSV format

    def remove_garbage(line):
        """
        Remove spaces, /, and \n characters from the line.

        :param line: Line currently being processed.
        :return: line: data without the unnecessary information.
        """

        line = re.sub(r'\s{2,}', '', line)
        line = re.sub(r'\n', '', line)
        # line 16 data ends with text fields. We need the space and not the comma. Everything else needs the comma.
        if re.search(r'^16,', line):
            line = re.sub(r'/', ' ', line)
        else:
            line = re.sub(r'/', ',', line)

        return line


    def extract_02_data(line):
        """
        The 02-line is the group header for the account numbers and transactions that follow. The data that follows all
        have to do with the single routing number from this group. Use the regex strings to find the necessary data from
        the 02 line. The rest of the data will be based on the as-of-date. The currency field is optional. If it is blank,
        the default is 'USD'. Routing number is not necessary for our current needs.

        :param line: The 02 line that was extracted from the parse_file function
        :return: file_date, routing_num, currency
        """
        # 02_line regex data
        as_of_date = r'^02,\w*,\w+,\w+,(\d+)'
        group_currency = r'^02,\w*,\w+,\d+,\d+,\d*,(\w*)'

        file_date = datetime.datetime.strptime(re.search(as_of_date, line).group(1), '%y%m%d').strftime('%m/%d/%Y')
        currency = re.search(group_currency, line).group(1)
        # If no value is specified, the default is 'USD'
        if currency == '':
            currency = 'USD'

        return file_date, currency

    def extract_03_data(line, currency_type):
        """
        The 03-line contains the account identifier and account summary information. Use the regex values to find the data
        that is found in the 03-line. The account_currency value will get individual currencies if the group has multiple
        account currencies. We currently only want available balance data. We can add more balances in the future if
        necessary.

        :param line: 03_line that has been cleanded during parse_file function
        :param currency_type: currency from the 02_line data
        :return:
        """

        account_number = r'^03,(\w+)'
        account_currency = r'^03,\w+,(\w*)'
        opening_available = r',040,((\+\d+)|(-\d+)|(\d+))'
        closing_available = r',045,((\+\d+)|(-\d+)|(\d+))'
        current_available = r',060,((\+\d+)|(-\d+)|(\d+))'

        # Account numbers can be alphanumeric.
        account_num = re.search(account_number, line).group(1)
        # If the account has a currency that is specific to that account, it will be displayed.
        currency = re.search(account_currency, line).group(1)
        # If there is not a currency value, the default is from the '02' line data.
        if currency == '':
            currency = currency_type
        if re.search(opening_available, line):
            open_avail = add_cents(re.search(opening_available, line).group(1))
        else:
            open_avail = 'Value Not Found'
        # Check for two different codes in the '03' line for closing/ending/current balance in account.
        if re.search(closing_available, line):
            close_avail = add_cents(re.search(closing_available, line).group(1))
        else:
            close_avail = 'Value Not Found'
            if close_avail == 'Value Not Found' and re.search(current_available, line):
                close_avail = add_cents(re.search(current_available, line).group(1))
            else:
                close_avail = 'Value Not Found'

        return account_num, currency, open_avail, close_avail


    def extract_16_data(line, look_up):
        """
        The 16-line contains all of the transaction information for the account found in the 03-line. The regexes collect
        the transaction data from the 16-line. The 16-line contains a comment field in the last position that contains
        information that is sometimes used, but the inconsistent use of commas make it challenging to collect. Instead of
        collecting the comments field, the whole line will be added to the output DF in a column.
        :param line:
        :param look_up:
        :return:
        """

        transaction_type_code = r'16,(\d+)'
        transaction_amount = r'16,\d+,(\d*)'

        # The funds type variable changes how the comments are searched for.
        funds_type = r'^16,\d+,\d*,([012SVDZ]*)'

        # Inconsistent use of commas means that the customer reference number and comments are grabbed together.
        cust_ref_num_txns_comment = r'^16,\d+,\d*,[012SVDZ]*,\w*,(.*)'
        # Type S means three additional fields: immediate, one-day, and more than one day availability.
        if funds_type == 'S':
            cust_ref_num_txns_comment = r'^16,\d+,\d*,[012SVDZ]*,\d+,\d+,\d+,\w*,(.*)'
        # Type V means the next two fields are value date (not-optional) and value time (optional).
        if funds_type == 'V':
            cust_ref_num_txns_comment = r'^16,\d+,\d*,[012SVDZ]*,\d+,\d*,\w*,(.*)'
        # Type D means there are multiple distributions, each with distribution day and amount.
        if funds_type == 'D':
            distributions = int(re.search(r'^16,\d+,\d*,[012SVDZ]*,(\d+)', line).group(1))
            # Day and Amount for every distribution
            total_distributions = r'\d+,\d+' * distributions
            cust_ref_num_txns_comment = r'^16,\d+,\d*,[012SVDZ]*,\w*,' + total_distributions + ',(.*)'

        txns_cust_ref_comments = re.sub(',', '', re.search(cust_ref_num_txns_comment, line).group(1).strip())

        # The transaction code needs to be converted to an int() for searching in the range function.
        txns_type_code = int(re.search(transaction_type_code, line).group(1))

        # The transaction code is used to look-up the transaction type in the look-up table.
        try:
            txns_type = look_up.loc[int(txns_type_code)]['TransDesc']
        except KeyError:
            txns_type = 'Unknown Value'

        # All transaction amounts come in as positive values, the transaction code identifies whether it was a Debit or
        # Credit.
        if txns_type_code in range(400, 700):
            txns_amount = add_cents(re.search(transaction_amount, line).group(1)) * -1
            debit_credit = 'Debit'
        else:
            txns_amount = add_cents(re.search(transaction_amount, line).group(1))
            debit_credit = 'Credit'

        return txns_amount, debit_credit, txns_type_code, txns_type, txns_cust_ref_comments


    def parse_file(df, look_up):
        """
        Separates the file into the lines that contain all the data. 02 lines contain Group Header information. 03 lines
        contain Account Identifier and Summary/Status information. 16 lines contain Transaction Detail information.

        :param df: text file containing all of the lines for processing.
        :param look_up: the look-up table for converting transaction codes to transaction descriptions
        :return: lines_03 and lines_16 are lists with their respective data from the file.
        """
        lines_03 = list()
        lines_16 = list()

        index = 0
        while index < len(df):
            line = df[index]
            if re.search('^02,', line):
                line_02 = line
                index += 1
                while index < len(df) and re.search('^88,', df[index]):
                    line_02 += df[index][3:]
                    index += 1
                line_02 = (remove_garbage(line_02))
                as_of_date, group_currency = extract_02_data(line_02)
                continue
            if re.search('^03,', line):
                line_03 = line
                index += 1
                while index < len(df) and re.search('^88,', df[index]):
                    line_03 += df[index][3:]
                    index += 1
                acc_num, acc_currency, open_bal, close_bal = extract_03_data(remove_garbage(line_03), group_currency)
                lines_03.append((acc_num, open_bal, close_bal, acc_currency, as_of_date))
                continue
            if re.search('^16,', line):
                line_16 = line
                index += 1
                while index < len(df) and re.search('^88,', df[index]):
                    line_16 += df[index][3:]
                    index += 1
                txns_amount, debit_credit, txns_type_code, txns_type, txns_cust_ref_comments = extract_16_data(
                    remove_garbage(line_16), look_up)
                lines_16.append((acc_num, acc_currency, txns_amount, debit_credit, txns_type_code,
                                 txns_type, as_of_date, txns_cust_ref_comments, remove_garbage(line_16)))
                continue
            index += 1

        return as_of_date, lines_03, lines_16
