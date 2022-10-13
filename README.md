[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **Deribit_inverse_BTC_options** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml

Name of Quantlet: 'Deribit_inverse_BTC_options'
Published in: 'Pricing and hedging inverse BTC options'
Description: 'We focus on the inverse BTC options traded in Deribit and provide codes to calibrate the BS, SV, SVJ, and SVCJ models and to calculate their deltas.'
Keywords: Bitcoin, Deribit, stocahstic volatility, jump, dynamic delta hedge, options
Author: Huei-Wen Teng, Wolfgang Haerdle
Submitted: 20220906

```

### MATLAB Code
```matlab

% filename: GitHub_calibrate_BS.m
% written by Huei-Wen Teng and updated on 2022/9/6
% Descriptions: This main file demonstrate how to calibrate the BS, SV,
% SVJ, and SVCJ using Deribit inverse options
% It also calcualte delta under the BS, SV, SVJ, SVCJ model.
clc; clear all; close all;

filename = 'Deribit_20220101.csv'; % Downloaded on 2022/8/31
fileID = fopen(filename);
C = textscan(fileID,'%q %q %q %q %q %q %q %q %q %q %q %q', 'HeaderLines', 1, 'Delimiter',',');
fclose(fileID);
q = str2double(C{1});
p = str2double(C{2});
s = string(C{3});
t = str2double(C{4});
d0 = string(C{5});
d = datenum(d0,'yyyy-mm-dd');
trade_seq = str2double(C{6});
trade_id = str2double(C{7});
iv = str2double(C{9})/100;
instrument_name = string(C{10});
temp = split(instrument_name, '-'); % split it into underlying, maturity, strike, cp
underlying = temp(:, 1);
maturity0 = temp(:, 2);   %
maturity = datenum(maturity0, 'ddmmmyy');
strike = str2double(temp(:, 3)); % make this a number
cp = temp(:,4);         % call or put
cp = char(cp);
index_price = str2double(C{11});
direction = string(C{12});
DTM = maturity - d;                     % days to maturity
tau = (maturity-d)/365;                 % year to maturity (365 days)
moneyness =  index_price ./ strike;     % moneyness k = S0/K
omega = -1*(cp=='P') + 1*(cp=='C');
nDay_max = max(DTM);

n = 50; % Monte Carlo sample size
U_base = unifrnd(0, 1, [2*n,  nDay_max]);
Z_base = normrnd(0, 1, [3*n, nDay_max]);

nDay_max = max(maturity-d);
contract = [omega (maturity-d) strike index_price];
[contract_base ia ic]= unique(contract, 'rows', 'sort');
n_base = length(ia);
omega_base = contract_base(:, 1);
maturity_base = contract_base(:, 2);    % t = T
strike_base = contract_base(:, 3);
index_price_base = contract_base(:, 4);
d_base = zeros(n_base, 1);              % t = 0

% Task 1: Calibrate the BS model
curr_model = 'BS';
param0 = 0.0267; % initial value
options = optimset('Display', 'iter', 'PlotFcns', @optimplotfval, 'TolFun', 1e-6, 'TolX', 1e-1);
fun = @(param)obj_fminsearch(param, curr_model, n, U_base, Z_base, p, index_price_base,  omega_base, d_base, maturity_base, strike_base, ic);
[param, fval, exitflag, output] = fminsearch(fun, param0, options);

% Task 2: calibrate the SV model
curr_model = 'SV';
param0 = [0.0009 -0.9970 0.0002 0.8706 0.0019 0.0015]; % initial value
options = optimoptions(@lsqnonlin, 'Algorithm', 'levenberg-marquardt', 'Display', 'iter');
fun = @(param)obj_lsqnonlin(param, curr_model, n, p, U_base, Z_base, index_price_base, omega_base, d_base, maturity_base, strike_base, ic);
lb = -5 * ones(6,1);
ub =  5 * ones(6,1);
lb(2) = -1;% -1 < rho <1
lb(3) = 0; % alpha > 0
lb(5) = 0; % V0 > 0
lb(6) = 0; % sigma_v > 0
up(2) = 1;% -1 < rho <1
up(4) = 1; % beta<1
[param, resnorm, residual, exitflag, output] = lsqnonlin(fun, param0, lb, ub,options);

% Task 3: calibrate the SVJ model
curr_model = 'SVJ';
param0 = [0.0003   -0.1705   -0.0000   -4.9991    0.0000    1.9627    0.2340    0.0035    0.0792]; % initial value
options = optimoptions(@lsqnonlin, 'Algorithm', 'levenberg-marquardt', 'Display', 'iter');
fun = @(param)obj_lsqnonlin(param, curr_model, n, p, U_base, Z_base, index_price_base, omega_base, d_base, maturity_base, strike_base, ic);
lb = -5 * ones(9,1);
ub =  5 * ones(9,1);
lb(2) = -1;% -1 < rho <1
lb(3) = 0; % alpha > 0
lb(5) = 0; % V0 > 0
lb(6) = 0; % sigma_v > 0
up(2) = 1;% -1 < rho <1
up(4) = 1; % beta<1
lb(7) = 0; % 0<lambda <1
lb(9) = 0; % sigma_y >0
ub(7) = 1; % 0<lambda <1
[param, resnorm, residual, exitflag, output] = lsqnonlin(fun, param0, lb, ub,options);

% Task 4: calibrate the SVCJ model
curr_model = 'SVCJ';
param0 = [-0.0002   -0.8922    0.0025   -2.0661    0.0025    0.0603    0.0067    0.1010   -1.0588    0.0212    0.0000]; % initial param0
options = optimoptions(@lsqnonlin, 'Algorithm', 'levenberg-marquardt', 'Display', 'iter');
fun = @(param)obj_lsqnonlin(param, curr_model, n, p, U_base, Z_base, index_price_base, omega_base, d_base, maturity_base, strike_base, ic);
lb = -5 * ones(11,1);
ub =  5 * ones(11,1);
lb(2) = -1;% -1 < rho <1
lb(3) = 0; % alpha > 0
lb(5) = 0; % V0 > 0
lb(6) = 0; % sigma_v > 0
up(2) = 1;% -1 < rho <1
up(4) = 1; % beta<1
lb(7) = 0; % 0<lambda <1
ub(7) = 1; % 0<lambda <1
lb(10) = 0; % sigma_y >0
lb(11) = 0; % mu_v >0
[param, resnorm, residual, exitflag, output] = lsqnonlin(fun, param0, lb, ub,options);

% Task 5: calcualte BS delta
omega = 1;
sig = 0.0267;
sig2 = sig*sig;
index_price = 5000;
strike = 4000;
r = 0;
y = 0;
strikehat = 1/strike;
tau = 100;
d1 = log(index_price/strike)/(sig*sqrt(tau))+(r-y+0.5*sig)*sqrt(tau);
d2 = d1-sig*sqrt(tau);
d3 = d2-sig *sqrt(tau);
delta = omega * (exp((sig2)*tau) * (1/index_price^2)*strike*normcdf(omega*d3));

% Task 6: calculate SV delta
omega = 1;
sig = 0.0267;
sig2 = sig*sig;
index_price = 5000;
strike = 4000;
r = 0;
y = 0;
strikehat = 1/strike;
tau = 100;

mu = 0.0009;  rho = -0.9970; alpha = 0.0002; beta = 0.8706; V0 = 0.0019;  sig_v = 0.0015;
n = 200; % Monte Carlo sample size
Z1 = normrnd(0, 1, [n, tau]);
Z2 = normrnd(0, 1, [n, tau]);
x1 = Z1; % noise for the return yt
x2 = rho* Z1 + sqrt(1-rho^2)*Z2; % noise for the volatiltiy Vt
V = zeros(n, tau);
t = 1;
V(:, 1) = alpha +  beta * V0 + sig_v * sqrt(V0) .* x2(:, t);
for t = 2: tau
    index = find( V(:, (t-1)) < 0 );
    if length(index) > 0
        V(index, (t-1)) = 0;
    end
    V(:, t) = alpha +  beta * V(:, (t-1)) + sig_v * sqrt(V(:, t-1)) .* x2(:, t);
end
Y = zeros(n, tau);
t = 1;
Y(:, t) = mu + sqrt(V0) * x1(:, t);
for t = 2: tau
    Y(:, t) = mu + sqrt(V(:, t-1)) .* x1(:, t);
end
S = index_price * exp(cumsum(Y, 2));
Shat = 1/index_price;
delta_temp =  exp(-r*tau) * omega* Shat^2 * max( omega *(index_price - strike), 0);
delta = mean(delta_temp);


% Task 7: calculate SVJ delta
omega = 1;
sig = 0.0267;
sig2 = sig*sig;
index_price = 5000;
strike = 4000;
r = 0;
y = 0;
strikehat = 1/strike;
tau = 100;
mu = 0.0009;  rho = -0.9970; alpha = 0.0002; beta = 0.8706; V0 = 0.0019;  sig_v = 0.0015;
n = 100;
U1 = unifrnd(0, 1, [n,  tau]);
Z1 = normrnd(0, 1, [n, tau]);
Z2 = normrnd(0, 1, [n, tau]);
Z3 = normrnd(0, 1, [n, tau]);

param = [0.0003   -0.1705   -0.0000   -4.9991    0.0000    1.9627    0.2340    0.0035    0.0792];
mu = param(1); rho = param(2); alpha = param(3); beta = param(4); V0 = param(5); sig_v = param(6); lambda = param(7); mu_y = param(8); sig_y = param(9);

J = get_Bernoulli(lambda, U1);
jump1 = ( mu_y ) + abs(sig_y) * Z1; % Jump for the return
x1 = Z2; % noise for the return yt
x2 = rho* Z2 + sqrt(1-rho^2)*Z3; % noise for the volatiltiy Vt
V = zeros(n, tau);
t = 1;
V(:, 1) = alpha +  beta * V0 + sig_v * sqrt(V0) .* x2(:, t);

for t = 2: tau
    index = find( V(:, (t-1)) < 0 );
    if length(index) > 0
        V(index, (t-1)) = 0;
    end
    V(:, t) = alpha +  beta * V(:, (t-1)) + sig_v * sqrt(V(:, t-1)) .* x2(:, t);
end
Y = zeros(n, tau);
t = 1;
Y(:, t) = mu + sqrt(V0) * x1(:, t) + jump1(:,t).*J(:, t);
for t = 2: tau
    Y(:, t) = mu + sqrt(V(:, t-1)) .* x1(:, t) + jump1(:,t).*J(:, t);
end
S = index_price * exp(cumsum(Y, 2));

Shat = 1/index_price;
delta_temp =  exp(-r*tau) * omega* Shat^2 * max( omega *(index_price - strike), 0);
delta = mean(delta_temp);

% Task 8: calculate SVCJ delta
omega = 1;
sig = 0.0267;
sig2 = sig*sig;
index_price = 5000;
strike = 4000;
r = 0;
y = 0;
strikehat = 1/strike;
tau = 100;

mu = 0.0009;  rho = -0.9970; alpha = 0.0002; beta = 0.8706; V0 = 0.0019;  sig_v = 0.0015;
n = 100; 
U1 = unifrnd(0, 1, [n,  tau]);
U2 = unifrnd(0, 1, [n,  tau]);
Z1 = normrnd(0, 1, [n, tau]);
Z2 = normrnd(0, 1, [n, tau]);
Z3 = normrnd(0, 1, [n, tau]);

param = [ -0.0002   -0.8922    0.0025   -2.0661    0.0025    0.0603    0.0067    0.1010   -1.0588    0.0212    0.0000];
mu = param(1);  rho = param(2); alpha = param(3); beta = param(4); V0 = param(5); sig_v = param(6); lambda = param(7); mu_y = param(8); rho_j = param(9); sig_y = param(10); mu_v = param(11);
J = get_Bernoulli(lambda, U1);
jump2 =  get_exp(1/mu_v, U2); 
jump1 = ( mu_y + rho_j * jump2 ) + abs(sig_y) * Z1; 
x1 = Z2; 
x2 = rho* Z2 + sqrt(1-rho^2)*Z3; 
V = zeros(n, tau);
t = 1;
V(:, 1) = alpha +  beta * V0 + sig_v * sqrt(V0) .* x2(:, t) + jump2(:, t).*J(:, t);

for t = 2: tau
    index = find( V(:, (t-1)) < 0 );
    if length(index) > 0
        V(index, (t-1)) = 0;
    end
    V(:, t) = alpha +  beta * V(:, (t-1)) + sig_v * sqrt(V(:, t-1)) .* x2(:, t) + jump2(:, t).*J(:, t);
end

Y = zeros(n, tau);
t = 1;
Y(:, t) = mu + sqrt(V0) * x1(:, t) + jump1(:,t).*J(:, t);
for t = 2: tau
    Y(:, t) = mu + sqrt(V(:, t-1)) .* x1(:, t) + jump1(:,t).*J(:, t);
end
S = index_price * exp(cumsum(Y, 2));

Shat = 1/index_price;
delta_temp =  exp(-r*tau) * omega* Shat^2 * max( omega *(index_price - strike), 0);
delta = mean(delta_temp);

```

automatically created on 2022-10-13