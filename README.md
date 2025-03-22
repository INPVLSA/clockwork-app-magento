### Magento module repository this app used in

https://github.com/INPVLSA/magento-clockwork

### Original clockwork-app repository

https://github.com/underground-works/clockwork-app

### This one Magento2 - specific

(doc raw)

### Dev preparation

1. Change .env.development
2. Change `origin` in `vite.config.js` to evade CORS error
3. `ln -s ../app/code/Inpvlsa/Clockwork/view/frontend/web/ dist`

### Run
```
npm run serve
```
