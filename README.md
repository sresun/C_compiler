# C_compiler

/* C compiler

Copyright 1972 Bell Telephone Laboratories, Inc. 

*/

init(s, t)
char s[]; {
	extern symbuf, namsiz;
	char symbuf[], sp[];
	int np[], i;

	i = namsiz;
	sp = symbuf;
	while(i--)
		if ((*sp++ = *s++)=='\0') --s;
	np = lookup();
	*np++ = 1;
	*np = t;
}

main(argc, argv)
int argv[]; {
	extern extdef, eof;
	extern fout, fin, nerror, tmpfil, xdflg;

	if(argc<4) {
		error("Arg count");
		exit(1);
	}
	if((fin=open(argv[1],0))<0) {
		error("Can't find %s", argv[1]);
		exit(1);
	}
	if((fout=creat(argv[2], 017))<0) {
		error("Can't create %s", argv[2]);
		exit(1);
	}
	tmpfil = argv[3];
	xdflg++;
	init("int", 0);
	init("char", 1);
	init("float", 2);
	init("double", 3);
	init("struct", 4);
	init("auto", 5);
	init("extern", 6);
	init("static", 7);
	init("goto", 10);
	init("return", 11);
	init("if", 12);
	init("while", 13);
	init("else", 14);
	init("switch", 15);
	init("case", 16);
	init("break", 17);
	init("continue", 18);
	init("do", 19);
	init("default", 20);
	xdflg = 0;
	while(!eof) {
		extdef();
		blkend();
	}
	flush();
	flshw();
	exit(nerror!=0);
}

lookup() {
	extern hshtab, hshsiz, pssiz, symbuf, xdflg;
	int hshtab[], symbuf[];
	extern hshlen, hshused, nwps;
	auto i, j, np[], sp[], rp[];

	i = 0;
	sp = symbuf;
	j = nwps;
	while(j--)
		i =+ *sp++ & 077577;
	if (i<0) i = -i;
	i =% hshsiz;
	i =* pssiz;
	while(*(np = &hshtab[i+4])) {
		sp = symbuf;
		j = nwps;
		while(j--)
			if ((*np++&077577) != *sp++) goto no;
		return(&hshtab[i]);
no:		if ((i =+ pssiz) >= hshlen) i = 0;
	}
	if(++hshused > hshsiz) {
		error("Symbol table overflow");
		exit(1);
	}
	rp = np = &hshtab[i];
	sp = symbuf;
	j = 4;
	while(j--)
		*np++ = 0;
	j = nwps;
	while(j--)
		*np++ = *sp++;
	*np = 0;
	if (xdflg)
		rp[4] =| 0200;		/* mark non-deletable */
	return(rp);
}

symbol() {
	extern peeksym, peekc, eof, line;
	extern csym, symbuf, namsiz, lookup, ctab, cval;
	int csym[];
	extern isn, mosflg, xdflg;
	auto b, c;
	char symbuf[], sp[], ctab[];

	if (peeksym>=0) {
		c = peeksym;
		peeksym = -1;
		if (c==20)
			mosflg = 0;
		return(c);
	}
	if (peekc) {
		c = peekc;
		peekc = 0;
	} else
		if (eof)
			return(0); else
			c = getchar();
loop:
	switch(ctab[c]) {

	case 125:	/* newline */
		line++;

	case 126:	/* white space */
		c = getchar();
		goto loop;

	case 0:		/* EOF */
		eof++;
		return(0);

	case 40:	/* + */
		return(subseq(c,40,30));

	case 41:	/* - */
		return(subseq(c,subseq('>',41,50),31));

	case 80:	/* = */
		if (subseq(' ',0,1)) return(80);
		c = symbol();
		if (c>=40 & c<=49)
			return(c+30);
		if (c==80)
			return(60);
		peeksym = c;
		return(80);

	case 63:	/* < */
		if (subseq(c,0,1)) return(46);
		return(subseq('=',63,62));

	case 65:	/* > */
		if (subseq(c,0,1)) return(45);
		return(subseq('=',65,64));

	case 34:	/* ! */
		return(subseq('=',34,61));

	case 43:	/* / */
		if (subseq('*',1,0))
			return(43);
com:
		c = getchar();
com1:
		if (c=='\0') {
			eof++;
			error("Nonterminated comment");
			return(0);
		}
		if (c=='\n')
			line++;
		if (c!='*')
			goto com;
		c = getchar();
		if (c!='/')
			goto com1;
		c = getchar();
		goto loop;

	case 120:	/* . */
	case 124:	/* number */
		peekc = c;
		switch(c=getnum(c=='0'? 8:10)) {
			case 25:		/* float 0 */
				c = 23;
				break;

			case 23:		/* float non 0 */
				cval = isn++;
		}
		return(c);

	case 122:	/* " */
		return(getstr());

	case 121:	/* ' */
		return(getcc());

	case 123:	/* letter */
		sp = symbuf;
		if (mosflg) {
			*sp++ = '.';
			mosflg = 0;
		}
		while(ctab[c]==123 | ctab[c]==124) {
			if (sp<symbuf+namsiz) *sp++ = c;
			c = getchar();
		}
		while(sp<symbuf+namsiz)
			*sp++ = '\0';
		peekc = c;
		csym = lookup();
		if (csym[0]==1) {	/* keyword */
			cval = csym[1];
			return(19);
		}
		return(20);

	case 127:	/* unknown */
		error("Unknown character");
		c = getchar();
		goto loop;

	}
	return(ctab[c]);
}

subseq(c,a,b) {
	extern peekc;

	if (!peekc)
		peekc = getchar();
	if (peekc != c)
		return(a);
	peekc = 0;
	return(b);
}
getstr() {
	extern isn, cval, strflg;
	auto c;
	char t[], d[];

	t = ".text";
	d = ".data";
	printf("%s;L%d:.byte ", (strflg?t:d), cval=isn++);
	while((c=mapch('"')) >= 0)
		printf("%o,", c);
	printf("0;.even;%s\n", (strflg?d:t));
	return(22);
}

getcc()
{
	extern cval, ncpw;
	auto c, cc;
	char cp[];

	cval = 0;
	cp = &cval;
	cc = 0;
	while((c=mapch('\'')) >= 0)
		if(cc++ < ncpw)
			*cp++ = c;
	if(cc>ncpw)
		error("Long character constant");
	return(21);
}

mapch(c)
{
	extern peekc, line;
	auto a;

	if((a=getchar())==c)
		return(-1);
	switch(a) {

	case '\n':
	case 0:
		error("Nonterminated string");
		peekc = a;
		return(-1);

	case '\\':
		switch (a=getchar()) {

		case 't':
			return('\t');

		case 'n':
			return('\n');

		case '0':
			return('\0');

		case 'r':
			return('\r');

		case '\n':
			line++;
			return('\n');
		}

	}
	return(a);
}

tree() {
	extern csym, ctyp, isn, fcval, peeksym, opdope, cp, cmst;
	int csym[], opdope[], cp[], cmst[];
	extern space, cval, ossiz, cmsiz, mosflg, osleft;
	double fcval;
	int space[];

	int op[], opst[20], pp[], prst[20], andflg, o,
		p, ps, os;

	osleft = ossiz;
	space = 0;
	*space++ = 0;
	op = opst;
	pp = prst;
	cp = cmst;
	*op = 200;		/* stack EOF */
	*pp = 06;
	andflg = 0;

advanc:
	switch (o=symbol()) {

	/* name */
	case 20:
		if (*csym==0)
			if((peeksym=symbol())==6) {	/* ( */
				*csym = 6;		/* extern */
				csym[1] = 020;		/* int() */
			} else {
				csym[1] = 030;		/* array */
				if (csym[2]==0)
					csym[2] = isn++;
			}
		*cp++ = block(2,20,csym[1],csym[3],*csym,0);
		if (*csym==6) {			/* external */
			o = 3;
			while(++o<8) {
				pblock(csym[o]);
				if ((csym[o]&077400) == 0)
					break;
			}
		} else
			pblock(csym[2]);
		goto tand;

	/* short constant */
	case 21:
	case21:
		*cp++ = block(1,21,ctyp,0,cval);
		goto tand;

	/* floating constant */
	case 23:
		*cp++ = block(1,23,3,0,cval);
		if (cval)		/* non-0 */
			printf(".data;L%d:%o;%o;%o;%o;.text\n",cval,fcval);
		goto tand;

	/* string constant: fake a static char array */
	case 22:
		*cp++ = block(3, 20, 031, 1, 7, 0, cval);

tand:
		if(cp>=cmst+cmsiz) {
			error("Expression overflow");
			exit(1);
		}
		if (andflg)
			goto syntax;
		andflg = 1;
		goto advanc;

	/* ++, -- */
	case 30:
	case 31:
		if (andflg)
			o =+ 2;
		goto oponst;

	/* ! */
	case 34:
	/* ~ */
	case 38:
		if (andflg)
			goto syntax;
		goto oponst;

	/* - */
	case 41:
		if (!andflg) {
			peeksym = symbol();
			if (peeksym==21) {
				peeksym = -1;
				cval = -cval;
				goto case21;
			}
			o = 37;
		}
		andflg = 0;
		goto oponst;

	/* & */
	/* * */
	case 47:
	case 42:
		if (andflg)
			andflg = 0; else
			if(o==47)
				o = 35;
			else
				o = 36;
		goto oponst;

	/* ( */
	case 6:
		if (andflg) {
			o = symbol();
			if (o==7)
				o = 101; else {
				peeksym = o;
				o = 100;
				andflg = 0;
			}
		}
	goto oponst;

	/* ) */
	/* ] */
	case 5:
	case 7:
		if (!andflg)
			goto syntax;
		goto oponst;

	case 39:	/* . */
		mosflg++;
		break;

	}
	/* binaries */
	if (!andflg)
		goto syntax;
	andflg = 0;

oponst:
	p = (opdope[o]>>9) & 077;
opon1:
	ps = *pp;
	if (p>ps | p==ps & (opdope[o]&0200)!=0) { /* right-assoc */
putin:
		switch (o) {

		case 6: /* ( */
		case 4: /* [ */
		case 100: /* call */
			p = 04;
		}
		if(op>=opst+20) {		/* opstack size */
			error("expression overflow");
			exit(1);
		}
		*++op = o;
		*++pp = p;
		goto advanc;
	}
	--pp;
	switch (os = *op--) {

	/* EOF */
	case 200:
		peeksym = o;
		return(*--cp);

	/* call */
	case 100:
		if (o!=7)
			goto syntax;
		build(os);
		goto advanc;

	/* mcall */
	case 101:
		*cp++ = block(0,0,0,0);	/* 0 arg call */
		os = 100;
		goto fbuild;

	/* ( */
	case 6:
		if (o!=7)
			goto syntax;
		goto advanc;

	/* [ */
	case 4:
		if (o!=5)
			goto syntax;
		build(4);
		goto advanc;
	}
fbuild:
	build(os);
	goto opon1;

syntax:
	error("Expression syntax");
	errflush(o);
	return(0);
}

scdeclare(kw)
{
	extern csym, paraml, parame, peeksym;
	int csym[], paraml[], parame[];
	int o;

	while((o=symbol())==20) {		/* name */
		if(*csym>0 & *csym!=kw)
			redec();
		*csym = kw;
		if(kw==8)  {		/* parameter */
			*csym = -1;
			if (paraml==0)
				paraml = csym;
			else
				*parame = csym;
			parame = csym;
		}
		if ((o=symbol())!=9)	/* , */
			break;
	}
	if(o==1 & kw!=8 | o==7 & kw==8)
		return;
syntax:
	decsyn(o);
}

tdeclare(kw, offset, mos)
{
	int o, elsize, ds[];
	extern xdflg, peeksym, mosflg, defsym, csym;
	int csym[], ssym[];

	if (kw == 4) {				/* struct */
		ssym = 0;
		ds = defsym;
		mosflg = mos;
		if ((o=symbol())==20) {		/* name */
			ssym = csym;
			o = symbol();
		}
		mosflg = mos;
		if (o != 6) {			/* ( */
			if (ssym==0)
				goto syntax;
			if (*ssym!=8)		/* class structname */
				error("Bad structure name");
			if (ssym[3]==0) {	/* no size yet */
				kw = 5;		/* deferred MOS */
				elsize = ssym;
			} else
				elsize = ssym[3];
			peeksym = o;
		} else {
			if (ssym) {
				if (*ssym)
					redec();
				*ssym = 8;
				ssym[3] = 0;
			}
			elsize = declist(4);
			if ((elsize&01) != 0)
				elsize++;
			defsym = ds;
			if ((o = symbol()) != 7)	/* ) */
				goto syntax;
			if (ssym)
				ssym[3] = elsize;
		}
	}
	mosflg = mos;
	if ((peeksym=symbol()) == 1) {		/* ; */
		peeksym = -1;
		mosflg = 0;
		return(offset);
	}
	do {
		offset =+ t1dec(kw, offset, mos, elsize);
		if (xdflg & !mos)
			return;
	} while ((o=symbol()) == 9);		/* , */
	if (o==1)
		return(offset);
syntax:
	decsyn(o);
}

t1dec(kw, offset, mos, elsize)
{
	int type, nel, defsym[], t1;
	extern defsym, mosflg;

	nel = 0;
	mosflg = mos;
	if ((t1=getype(&nel)) < 0)
		goto syntax;
	type = 0;
	do
		type = type<<2 | (t1 & 03);
	while(t1 =>> 2);
	t1 = type<<3 | kw;
	if (defsym[1] & defsym[1]!=t1)
		redec();
	defsym[1] = t1;
	defsym[3] = elsize;
	elsize = length(defsym);
	if (mos) {
		if (*defsym)
			redec();
		else
			*defsym = 4;
		if ((offset&1)!=0 & elsize!=1)
			offset++;
		defsym[2] = offset;
	} else
		if (*defsym == 0)
			*defsym = -2;		/* default auto */
	if (nel==0)
		nel = 1;
	defsym[8] = nel;
syntax:
	return(nel*elsize);
}

getype(pnel)
int pnel[];
{
	int o, type;
	extern cval, peeksym, xdflg, defsym, csym, pssiz;
	int defsym[], csym[];

	switch(o=symbol()) {

	case 42:					/* * */
		return(getype(pnel)<<2 | 01);

	case 6:						/* ( */
		type = getype(pnel);
		if ((o=symbol()) != 7)			/* ) */
			goto syntax;
		goto getf;

	case 20:					/* name */
		defsym = csym;
		type = 0;
	getf:
		switch(o=symbol()) {

		case 6:					/* ( */
			if (xdflg) {
				xdflg = 0;
				o = defsym;
				scdeclare(8);
				defsym = o;
				xdflg++;
			} else
				if ((o=symbol()) != 7)	/* ) */
					goto syntax;
			type = type<<2 | 02;
			goto getf;

		case 4:					/* [ */
			if ((o=symbol()) != 5) {	/* ] */
				if (o!=21)		/* const */
					goto syntax;
				*pnel = cval;
				if ((o=symbol())!=5)
					goto syntax;
			}
			type = type<<2 | 03;
			goto getf;
		}
		peeksym = o;
		return(type);
	}
syntax:
	decsyn(o);
	return(-1);
}

decsyn(o)
{
	error("Declaration syntax");
	errflush(o);
}

redec()
{
	extern csym;
	int csym[];

	error("%p redeclared", &csym[4]);
}

/* storage */

regtab 0;
efftab 1;
cctab 2;
sptab 3;
symbuf[4];
pssiz 9;
namsiz 8;
nwps 4;
hshused;
hshsiz 100;
hshlen 900;	/* 9*hshsiz */
hshtab[900];
space;
cp;
cmsiz 40;
cmst[40];
ctyp;
isn 1;
swsiz 120;
swtab[120];
swp;
contlab;
brklab;
deflab;
nreg 4;
nauto;
stack;
peeksym 0177777;
peekc;
eof;
line 1;
defsym;
xdflg;
csym;
cval;
fcval 0;	/* a double number */
fc1 0;
fc2 0;
fc3 0;
ncpw 2;
nerror;
paraml;
parame;
tmpfil;
strflg;
ossiz 250;
osleft;
mosflg;
debug 0;


```

c01.c

'''
build(op) {
	extern cp[], cvtab, opdope[], maprel[];
	auto p1[], t1, d1, p2[], t2, d2, p3[], t3, d3, t;
	auto d, dope, leftc, cvn, pcvn;
	char cvtab[];

	if (op==4)  {		/* [] */
		build(40);	/* + */
		op = 36;	/* * */
	}
	dope = opdope[op];
	if ((dope&01)!=0) {	/* binary */
		p2 = disarray(*--cp);
		t2 = p2[1];
		chkfun(p2);
		d2 = p2[2];
		if (*p2==20)
			d2 = 0;
	}
	p1 = disarray(*--cp);
	if (op!=100 & op!=35)		/* call, * */
		chkfun(p1);
	t1 = p1[1];
	d1 = p1[2];
	if (*p1==20)
		d1 = 0;
	pcvn = 0;
	switch (op) {

	/* : */
	case 8:
		if (t1!=t2)
			error("Type clash in conditional");
		t = t1;
		goto nocv;

	/* , */
	case 9:
		*cp++ = block(2, 9, 0, 0, p1, p2);
		return;

	/* ? */
	case 90:
		if (*p2!=8)
			error("Illegal conditional");
		t = t2;
		goto nocv;

	/* call */
	case 100:
		if ((t1&030) != 020)
			error("Call of non-function");
		*cp++ = block(2,100,decref(t1),24,p1,p2);
		return;

	/* * */
	case 36:
		if (*p1==35 | *p1==29) {	/* & unary */
			*cp++ = p1[3];
			return;
		}
		if (*p1!=20 & d1==0)
			d1 = 1;
		if ((t1&030) == 020)		/* function */
			error("Illegal indirection");
		*cp++ = block(1,36,decref(t1),d1,p1);
		return;

	/* & unary */
	case 35:
		if (*p1==36) {			/* * */
			*cp++ = p1[3];
			return;
		}
		if (*p1==20) {
			*cp++ = block(1,p1[3]==5?29:35,incref(t1),1,p1);
			return;
		}
		error("Illegal lvalue");
		break;

	case 43:	/* / */
	case 44:	/* % */
	case 73:	/* =/ */
	case 74:	/* =% */
		d1++;
		d2++;

	case 42:	/* * */
	case 72:	/* =* */
		d1++;
		d2++;
		break;

	case 30:	/* ++ -- pre and post */
	case 31:
	case 32:
	case 33:
		chklval(p1);
		*cp++ = block(2,op,t1,max(d1,1),p1,plength(p1));
		return;

	case 39:	/* . (structure ref) */
	case 50:	/* -> (indirect structure ref) */
		if (p2[0]!=20 | p2[3]!=4)		/* not mos */
			error("Illegal structure ref");
		*cp++ = p1;
		t = t2;
		if ((t&030) == 030)	/* array */
			t = decref(t);
		setype(p1, t);
		if (op==39)		/* is "." */
			build(35);	/* unary & */
		*cp++ = block(1,21,7,0,p2[5]);
		build(40);		/* + */
		if ((t2&030) != 030)	/* not array */
			build(36);	/* unary * */
		return;
	}
	if ((dope&02)!=0)		/* lvalue needed on left? */
		chklval(p1);
	if ((dope&020)!=0)		/* word operand on left? */
		chkw(p1);
	if ((dope&040)!=0)		/* word operand on right? */
		chkw(p2);
	if ((dope&01)==0) {		/* unary op? */
		*cp++ = block(1,op,t1,max(d1,1),p1);
		return;
	}
	if (t2==7) {
		t = t1;
		p2[1] = 0;	/* no int cv for struct */
		t2 = 0;
		goto nocv;
	}
	cvn = cvtab[11*lintyp(t1)+lintyp(t2)];
	leftc = cvn&0100;
	t = leftc? t2:t1;
	if (op==80 & t1!=4 & t2!=4) {	/* = */
		t = t1;
		if (leftc | cvn!=1)
			goto nocv;
	}
	if (cvn =& 077) {
		if (cvn==077) {
	illcv:
			error("Illegal conversion");
			goto nocv;
		}
		if (cvn>4 & cvn<10) {		/* ptr conv */
			t = 0;			/* integer result */
			cvn = 0;
			if ((dope&04)!=0)	/* relational? */
				goto nocv;
			if (op!=41)	/* - */
				goto illcv;
			pcvn = cvn;
			goto nocv;
		}
		if (leftc) {
			if ((dope&010) != 0) {	/* =op */
				if (cvn == 1) {
					leftc = 0;
					cvn = 8;
					t = t1;
					goto rcvt;
				} else
					goto illcv;
			}
			d1 = (p1=convert(p1, t, d1, cvn, plength(p2)))[2];
		} else {
		rcvt:
			d2 = (p2=convert(p2, t, d2, cvn, plength(p1)))[2];
		}
nocv:;		}
	if (d1==d2)
		d = d1+1; else
		d = max(d1,d2);
	if ((dope&04)!=0) {		/* relational? */
		if (op>61 & t>=010)
			op =+ 4;	  /* ptr relation */
		t = 0;		/* relational is integer */
	}
	*cp++ = optim(block(2,op,t,d,p1,p2));
	if (pcvn) {
		p1 = *--cp;
		*cp++ = block(1,50+pcvn,0,d,p1);
	}
	return;
	*cp++ = block(1,op,t1,d1==0?1:d1,p1);
}

setype(p, t)
int p[];
{
	int p1[];

	if ((p[1]&07) != 4)		/* not structure */
		return;
	p[1] = t;
	switch(*p) {

	case 29:		/* & */
	case 35:
		setype(p[3], decref(t));
		return;

	case 36:		/* * */
		setype(p[3], incref(t));
		return;

	case 40:		/* + */
		setype(p[4], t);
	}
}

chkfun(p)
int p[];
{
	if ((p[1]&030)==020)	/* func */
		error("Illegal use of function");
}

optim(p)
int p[];
{
	int p1[], p2[], t;

	if (*p != 40)				/* + */
		return(p);
	p1 = p[3];
	p2 = p[4];
	if (*p1==21) {				/* const */
		t = p1;
		p1 = p2;
		p2 = t;
	}
	if (*p2 != 21)				/* const */
		return(p);
	if ((t=p2[3]) == 0)			/* const 0 */
		return(p1);
	if (*p1!=35 & *p1!=29)			/* not & */
		return(p);
	p2 = p1[3];
	if (*p2!=20) {				/* name? */
		error("C error (optim)");
		return(p);
	}
	p2[4] =+ t;
	return(p1);
}

disarray(p)
int p[];
{
	extern cp;
	int t, cp[];

	if (((t = p[1]) & 030)!=030 | p[0]==20&p[3]==4)	/* array & not MOS */
		return(p);
	p[1] = decref(t);
	*cp++ = p;
	build(35);				/* add & */
	return(*--cp);
}

convert(p, t, d, cvn, len)
int p[];
{
	int c, p1[];

	if (*p==21) {		/* constant */
		c = p[3];
		switch(cvn) {

		case 4:		/* int -> double[] */
			c =<< 1;

		case 3:		/* int -> float[] */
			c =<< 1;

		case 2:		/* int -> int[] */
			c =<< 1;
			p[3] = c;
			return(p);

		case 10:	/* i -> s[] */
			p[3] = c*len;
			return(p);
		}
	}
	if (cvn==10)			/* i -> s[]; retrun i*len */
		return(block(2,42,t,d+2,p,block(1,21,0,0,len)));
	return(block(1, 50+cvn, t, max(1,d), p));
}

chkw(p)
int p[]; {
	extern error;
	auto t;

	if ((t=p[1])>1 & t<=07)
		error("Integer operand required");
	return;
}

lintyp(t)
{
	if (t<=07)
		return(t);
	if ((t&037)==t)
		return((t&07)+5);
	return(10);
}

error(s, p1, p2, p3, p4, p5, p6) {
	extern line, fout, nerror;
	int f;

	nerror++;
	flush();
	f = fout;
	fout = 1;
	printf("%d: ", line);
	printf(s, p1, p2, p3, p4, p5, p6);
	putchar('\n');
	fout = f;
}

block(n, op, t, d, p1,p2,p3)
int p1[],p2[],p3[]; {
	int p[], ap[], space[];
	extern space;

	ap = &op;
	n =+ 3;
	p = space;
	while(n--)
		pblock(*ap++);
	return(p);
}

pblock(p)
{
	extern space, osleft;
	int space[];

	*space++ = p;
	if (--osleft<=0) {
		error("Expression overflow");
		exit(1);
	}
}

chklval(p)
int p[]; {
	extern error;

	if (*p!=20 & *p !=36)
		error("Lvalue required");
}

max(a, b)
{
	if (a>b)
		return(a);
	return(b);
}

'''

c02.c

'''

extdef() {
	extern eof, cval, defsym;
	extern csym, strflg, xdflg, peeksym, fcval;
	int o, c, cs[], type, csym[], width, nel, ninit, defsym[];
	char s[];
	float sf;
	double fcval;

	if(((o=symbol())==0) | o==1)	/* EOF */
		return;
	type = 0;
	if (o==19) {			/* keyword */
		if ((type=cval)>4)
			goto syntax;	/* not type */
	} else {
		if (o==20)
			csym[4] =| 0200;	/* remember name */
		peeksym = o;
	}
	defsym = 0;
	xdflg++;
	tdeclare(type, 0, 0);
	if (defsym==0)
		return;
	*defsym = 6;
	cs = &defsym[4];
	printf(".globl	%p\n", cs);
	strflg = 1;
	xdflg = 0;
	type = defsym[1];
	if ((type&030)==020) {		/* a function */
		printf(".text\n%p:\nmov r5,-(sp); mov sp,r5\n", cs);
		declist(0);
		strflg = 0;
		c = 0;
		if ((peeksym=symbol())!=2) {	/* { */
			blkhed();
			c++;
		}
		statement(1);
		retseq();
		if (c)
			blkend();
		return;
	}
	width = length(defsym);
	if ((type&030)==030)	/* array */
		width = plength(defsym);
	nel = defsym[8];
	ninit = 0;
	if ((peeksym=symbol()) == 1) {	/* ; */
		printf(".comm	%p,%o\n", &defsym[4], nel*width);
		peeksym = -1;
		return;
	}
	printf(".data\n%p:", &defsym[4]);
loop:	{
		ninit++;
		switch(o=symbol()) {
	
		case 22:			/* string */
			if (width!=2)
				bxdec();
			printf("L%d\n", cval);
			break;
	
		case 41:			/* - const */
			if ((o=symbol())==23) {	/* float */
				fcval = -fcval;
				goto case23;
			}
			if (o!=21)
				goto syntax;
			cval = -cval;
	
		case 21:			/* const */
			if (width==1)
				printf(".byte ");
			if (width>2) {
				fcval = cval;
				goto case23;
			}
			printf("%o\n", cval);
			break;
	
		case 20:			/* name */
			if (width!=2)
				bxdec();
			printf("%p\n", &csym[4]);
			break;
	
		case 23:
		case23:
			if (width==4) {
				sf = fcval;
				printf("%o;%o\n", sf);
				break;
			}
			if (width==8) {
				printf("%o;%o;%o;%o\n", fcval);
				break;
			}
			bxdec();
			break;
	
		default:
			goto syntax;
	
		}
	} if ((o=symbol())==9) goto loop;	/* , */
	if (o==1) {			/* ; */
	done:
		if (ninit<nel)
			printf(".=.+%d.\n", (nel-ninit)*width);
		return;
	}
syntax:
	error("External definition syntax");
	errflush(o);
	statement(0);
}

bxdec()
{
	error("Inconsistent external initialization");
}

statement(d) {
	extern symbol, error, blkhed, eof, peeksym;
	extern blkend, csym[], rcexpr, block[], tree[], regtab[];
	extern retseq, jumpc, jump, label, contlab, brklab, cval;
	extern swp[], isn, pswitch, peekc;
	extern efftab[], deflab, errflush, swtab[], swsiz, branch;

	int o, o1, o2, o3, np[];

stmt:
	switch(o=symbol()) {

	/* EOF */
	case 0:
		error("Unexpected EOF");
	/* ; */
	case 1:
	/* } */
	case 3:
		return;

	/* { */
	case 2: {
		if(d)
			blkhed();
		while (!eof) {
			if ((o=symbol())==3)	/* } */
				goto bend;
			peeksym = o;
			statement(0);
		}
		error("Missing '}'");
	bend:
		return;
	}

	/* keyword */
	case 19:
		switch(cval) {

		/* goto */
		case 10:
			np = tree();
			if ((np[1]&030)!=030)	/* not array */
				np = block(1, 36, 1, np[2]+1, np);
			rcexpr(block(1,102,0,0,np), regtab);
			goto semi;

		/* return */
		case 11:
			if((peeksym=symbol())==6)	/* ( */
				rcexpr(block(1,110,0,0,pexpr()), regtab);
			retseq();
			goto semi;

		/* if */
		case 12:
			jumpc(pexpr(), o1=isn++, 0);
			statement(0);
			if ((o=symbol())==19 & cval==14) {  /* else */
				o2 = isn++;
				(easystmt()?branch:jump)(o2);
				label(o1);
				statement(0);
				label(o2);
				return;
			}
			peeksym = o;
			label(o1);
			return;

		/* while */
		case 13:
			o1 = contlab;
			o2 = brklab;
			label(contlab = isn++);
			jumpc(pexpr(), brklab=isn++, 0);
			o3 = easystmt();
			statement(0);
			(o3?branch:jump)(contlab);
			label(brklab);
			contlab = o1;
			brklab = o2;
			return;

		/* break */
		case 17:
			if(brklab==0)
				error("Nothing to break from");
			jump(brklab);
			goto semi;

		/* continue */
		case 18:
			if(contlab==0)
				error("Nothing to continue");
			jump(contlab);
			goto semi;

		/* do */
		case 19:
			o1 = contlab;
			o2 = brklab;
			contlab = isn++;
			brklab = isn++;
			label(o3 = isn++);
			statement(0);
			label(contlab);
			contlab = o1;
			if ((o=symbol())==19 & cval==13) { /* while */
				jumpc(tree(), o3, 1);
				label(brklab);
				brklab = o2;
				goto semi;
			}
			goto syntax;

		/* case */
		case 16:
			if ((o=symbol())!=21) {	/* constant */
				if (o!=41)	/* - */
					goto syntax;
				if ((o=symbol())!=21)
					goto syntax;
				cval = - cval;
			}
			if ((o=symbol())!=8)	/* : */
				goto syntax;
			if (swp==0) {
				error("Case not in switch");
				goto stmt;
			}
			if(swp>=swtab+swsiz) {
				error("Switch table overflow");
			} else {
				*swp++ = isn;
				*swp++ = cval;
				label(isn++);
			}
			goto stmt;

		/* switch */
		case 15:
			o1 = brklab;
			brklab = isn++;
			np = pexpr();
			if (np[1]>1 & np[1]<07)
				error("Integer required");
			rcexpr(block(1,110,0,0,np), regtab);
			pswitch();
			brklab = o1;
			return;

		/* default */
		case 20:
			if (swp==0)
				error("Default not in switch");
			if ((o=symbol())!=8)	/* : */
				goto syntax;
			deflab = isn++;
			label(deflab);
			goto stmt;
		}

		error("Unknown keyword");
		goto syntax;

	/* name */
	case 20:
		if (peekc==':') {
			peekc = 0;
			if (csym[0]>0) {
				error("Redefinition");
				goto stmt;
			}
			csym[0] = 7;
			csym[1] = 030;	/* int[] */
			if (csym[2]==0)
				csym[2] = isn++;
			label(csym[2]);
			goto stmt;
		}
	}

	peeksym = o;
	rcexpr(tree(), efftab);
	goto semi;

semi:
	if ((o=symbol())!=1)		/* ; */
		goto syntax;
	return;

syntax:
	error("Statement syntax");
	errflush(o);
	goto stmt;
}

pexpr()
{
	auto o, t;

	if ((o=symbol())!=6)	/* ( */
		goto syntax;
	t = tree();
	if ((o=symbol())!=7)	/* ) */
		goto syntax;
	return(t);
syntax:
	error("Statement syntax");
	errflush(o);
	return(0);
}

pswitch() {
	extern swp[], isn, swtab[], printf, deflab, statement, brklab;
	extern label;
	int sswp[], dl, cv, swlab;

	sswp = swp;
	if (swp==0)
		swp = swtab;
	swlab = isn++;
	printf("jsr	pc,bswitch; l%d\n", swlab);
	dl = deflab;
	deflab = 0;
	statement(0);
	if (!deflab) {
		deflab = isn++;
		label(deflab);
	}
	printf("L%d:.data;L%d:", brklab, swlab);
	while(swp>sswp & swp>swtab) {
		cv = *--swp;
		printf("%o; l%d\n", cv, *--swp);
	}
	printf("L%d; 0\n.text\n", deflab);
	deflab = dl;
	swp = sswp;
}

blkhed()
{
	extern symbol, cval, peeksym, paraml[], parame[];
	extern error, rlength, setstk, defvec, isn, defstat;
	extern stack, hshtab[], hshsiz, pssiz;
	int al, pl, cs[], hl, t[];

	declist(0);
	stack = al = 0;
	pl = 4;
	while(paraml) {
		*parame = 0;
		paraml = *(cs = paraml);
		if (cs[1]==2)		/* float args -> double */
			cs[1] = 3;
		cs[2] = pl;
		*cs = 10;
		if ((cs[1]&030) == 030)		/* array */
			cs[1] =- 020;		/* set ref */
		pl =+ rlength(cs);
	}
	cs = hshtab;
	hl = hshsiz;
	while(hl--) {
	    if (cs[4]) {
		if (cs[0]>1 & (cs[1]&07)==05) {  /* referred structure */
			t = cs[3];
			cs[3] = t[3];
			cs[1] = cs[1]&077770 | 04;
		}
		switch(cs[0]) {

		/* sort unmentioned */
		case -2:
			cs[0] = 5;		/* auto */

		/* auto */
		case 5:
			al =- trlength(cs);
			cs[2] = al;
			break;

		/* parameter */
		case 10:
			cs[0] = 5;
			break;

		/* static */
		case 7:
			cs[2] = isn;
			printf(".bss; L%d: .=.+%o; .text\n",
				isn++, trlength(cs));
			break;

		}
	    }
	    cs = cs+pssiz;
	}
	setstk(al);
}

blkend() {
	extern hshtab[], hshsiz, pssiz, hshused, debug;
	auto i, hl;

	i = 0;
	hl = hshsiz;
	while(hl--) {
		if(hshtab[i+4]) {
if (debug)
if (hshtab[i]!=1)
error("%p	%o	%o	%o	%o	%o",
	&hshtab[i+4],
	hshtab[i],
	hshtab[i+1],
	hshtab[i+2],
	hshtab[i+3],
	hshtab[i+8]);
			if (hshtab[i]==0)
				error("%p undefined", &hshtab[i+4]);
			if((hshtab[i+4]&0200)==0) {	/* not top-level */
				hshtab[i+4] = 0;
				hshused--;
			}
		}
		i =+ pssiz;
	}
}

errflush(o) {
	extern symbol, peeksym, eof;

	while(o>3)	/* ; { } */
		o = symbol();
	peeksym  = o;
}

declist(mosflg)
{
	extern peeksym, csym[], cval;
	auto o, offset;

	offset = 0;
	while((o=symbol())==19 & cval<10)
		if (cval<=4)
			offset = tdeclare(cval, offset, mosflg);
		else
			scdeclare(cval);
	peeksym = o;
	return(offset);
}

easystmt()
{
	extern peeksym, peekc, cval;

	if((peeksym=symbol())==20)	/* name */
		return(peekc!=':');	 /* not label */
	if (peeksym==19) {		/* keyword */
		switch(cval)

		case 10:	/* goto */
		case 11:	/* return */
		case 17:	/* break */
		case 18:	/* continue */
			return(1);
		return(0);
	}
	return(peeksym!=2);		/* { */
}

branch(lab)
{
	printf("br	L%d\n", lab);
}

'''


end


'''
