  ```javascript

  // Go into tx, look for transaction, look for signatures, and grab the first item ([0]).
  // If any of those steps are missing, don't crash—just return undefined. If it does return undefined, fall back and look for tx.signature instead."
  const signature = tx?.transaction?.signatures?.[0] ?? tx?.signature;

  // check if isArray type ?
  const transactions = Array.isArray(req.body) ? req.body : [req.body];

  // if wanna print the whole JSON which console.log wont because of the depth issue use below :
  console.dir(transactions, { depth: null, colors: true });

  

  ```
