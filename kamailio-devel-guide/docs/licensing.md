# Licensing #

Most of &kamailio; source code is licensed under GPLv2. New development to be introduced in the public repository
should have a GPLv2-compatible license.
the copyright for major developments is developer's choice, either the developer, the company is working for or
someone else.

Note that starting with v3.0.0, the contributions done to core components and main modules (tm, sl, auth) have to
be done under BSD license. This was decided when integration of Kamailio and SER projects started in order to avoid
conflicts between original developers and new ones.

If the code includes parts having other license make sure it is compatible with the GPLv2 and the way it is used now
does not violate the original license and the copyright.

Each C source file that is added under GPLv2 must include a header as follows:

    ...
    /**
    * Copyright (C) YEARS OWNER
    *
    * This file is part of Kamailio, a free SIP server.
    *
    * kamailio is free software; you can redistribute it and/or modify
    * it under the terms of the GNU General Public License as published by
    * the Free Software Foundation; either version 2 of the License, or
    * (at your option) any later version
    *
    * kamailio is distributed in the hope that it will be useful,
    * but WITHOUT ANY WARRANTY; without even the implied warranty of
    * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    * GNU General Public License for more details.
    *
    * You should have received a copy of the GNU General Public License 
    * along with this program; if not, write to the Free Software 
    * Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
    */
   ...

Each C source file that is added under BSD license must include a header
as follows:

    ...
    /**
    * Copyright (C) YEARS OWNER
    *
    * This file is part of Kamailio, a free SIP server.
    *
    * Permission to use, copy, modify, and distribute this software for any
    * purpose with or without fee is hereby granted, provided that the above
    * copyright notice and this permission notice appear in all copies.
    *
    * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
    * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
    * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
    * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
    * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
    * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
    * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
    */
   ...

YEARS and OWNER to be replaced appropriately.
